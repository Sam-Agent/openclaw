# Hermes Agent 代码分析

> 对 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) v0.13.0 的源码层面分析。
> 所有文件路径相对于上游仓库根目录；行号基于研究当日（2026-05-16）的 `main` 分支浅克隆。

## 1. 仓库总览

### 1.1 顶层结构

```
hermes-agent/
├── run_agent.py              # 主入口：AIAgent 类与回合循环（~16,360 行）
├── cli.py                    # TUI 入口与 Slash 命令分发（~14,166 行）
├── hermes_bootstrap.py       # Windows UTF-8 stdio 引导
├── hermes_constants.py       # ~/.hermes 主目录常量
├── hermes_state.py           # SQLite 会话存储（~2,966 行）
├── hermes_logging.py         # 日志封装
├── hermes_time.py            # 时区/时间工具
├── model_tools.py            # 工具定义聚合 + 异步桥接（~856 行）
├── toolsets.py               # Toolset 定义与解析（~855 行）
├── toolset_distributions.py  # Platform→toolset 默认套餐
├── agent/                    # 核心 agent 库（adapter / curator / memory / context）
├── tools/                    # 内置工具集合（terminal、browser、file、mcp …）
├── gateway/                  # 多平台消息网关
├── hermes_cli/               # 命令行子命令实现（setup、auth、kanban…）
├── skills/                   # 内置 Skills 包（25 个分类目录）
├── providers/                # Provider 基类
├── plugins/                  # 第三方插件机制
├── cron/                     # 定时任务调度器
├── acp_adapter/, acp_registry/   # Agent Client Protocol (IDE 集成)
├── tui_gateway/, ui-tui/     # React+Ink TUI 前端 + Python JSON-RPC 后端
├── web/, website/            # 仪表盘 / 官网
└── tests/                    # pytest 测试套件
```

### 1.2 关键配置文件

- `pyproject.toml`：所有直接依赖**精确钉版**（针对 2026-05 mistralai 2.4.6 Mini Shai-Hulud 蠕虫事件的供应链防御）。Provider/Backend 相关依赖通过 `[project.optional-dependencies]` extras + `tools/lazy_deps.py` 按需懒装。
- `uv.lock`：与 `pyproject.toml` 同步的传递依赖锁定文件，必须随精确钉版一同更新。
- `flake.nix` / `flake.lock`：Nix 包装。
- `docker-compose.yml` + `docker/`：容器化部署。
- `cli-config.yaml.example`：用户配置样例。

### 1.3 入口点

| 命令 | 入口 |
|------|------|
| `hermes` / `hermes-agent` | `hermes_cli/main.py:1-44` → `cli.py` |
| `hermes-acp` | `acp_adapter/entry.py:237-287` |
| `python -m gateway.run` | `gateway/run.py:1175` `GatewayRunner` |
| `batch_runner.py` | 批量轨迹生成 |
| `mcp_serve.py` | 将 Hermes 自身暴露为 MCP server |
| `cron/scheduler.py` | 定时任务后台进程 |

每个入口的第一行 `import hermes_bootstrap` 是强制约定：保证 Windows 控制台 UTF-8、防止 cp1252 编码崩溃（`hermes_bootstrap.py:74-127`）。

---

## 2. Agent 主循环：`run_agent.AIAgent`

### 2.1 类定位

`AIAgent` 类位于 `run_agent.py:1098` 起，是整个项目的事实核心。它实现：

- 与 LLM 的同步调用（chat completions / responses API / Anthropic Messages / Codex / Gemini 等）
- 工具调用循环（含并发与回写）
- 上下文压缩调度
- 记忆读写、Skills 装载、Trajectory 持久化
- 失败分类、重试、Provider 故障切换
- 流式输出回调、中断检测、子代理委派

### 2.2 单回合执行管线（`run_conversation`）

主循环位于 `run_agent.py:12526-15285`，结构如下：

| 阶段 | 行号 | 行为 |
|------|------|------|
| 初始化 | 12111-12487 | 追加 user 消息、加载记忆、重置重试计数、构造 `IterationBudget` |
| 系统提示词缓存 | 12298-12347 | 单会话内一次构建 `_cached_system_prompt`，三层结构（见 §2.5），首次后逐回合复用以维持 prefix cache |
| Preflight 压缩 | 12350-12416 | 加载历史若超阈值，调用 `_compress_context()`（最多 3 轮迭代） |
| 主回合循环 | 12526-15285 | 见 §2.3 |
| 回合后处理 | 15439-15980 | 外部记忆同步、记忆/Skill nudge 检查、后台 curator fork |

### 2.3 主回合循环逐步

```
while IterationBudget 未耗尽:
  ├── 检查中断（12531-12536）
  ├── 消费 budget（12547-12551）
  ├── step_callback（网关挂钩，12554-12579）
  ├── 排空 /steer 指令转为 tool message（12599-12635）
  ├── 修复 message 序列 / sanitize tool_call JSON（12637-12668）
  ├── 把 memory/plugin 注入到 *user* 消息而非系统提示（12670-12693）
  ├── 应用 Anthropic cache_control 标记（12749-12754）
  ├── 标准化空白以最大化 prefix 命中（12772-12803）
  ├── 调用 _interruptible_streaming_api_call（8045） 或 _interruptible_api_call（7727）
  │     → 实际请求在 worker 线程内发出，主线程仍可响应 Ctrl-C
  ├── 失败分类 _classify_api_error（9267）：
  │     - 429/速率 → 退避（jittered_backoff）
  │     - 401 → 凭证池轮换
  │     - context-window 4xx → 触发压缩并重试
  │     - quota → fallback provider chain
  ├── 解析响应 → 提取 tool calls
  ├── 并发/串行调度 _execute_tool_calls（10860）
  └── 把 tool 结果追加到 messages；若仍有 tool call 则继续，否则 break
```

### 2.4 工具调度

- `_execute_tool_calls()` (`run_agent.py:10860`)：先用 `_should_parallelize_tool_batch`（396）与 `_extract_parallel_scope_path`（440）判断该批工具是否**互不写同一路径**且全部为只读 / 路径不重叠。
  - 并发路径：11014 起，通过 `concurrent.futures.ThreadPoolExecutor` 并发执行。
  - 串行路径：11422 起，按顺序执行并实时 UI 渲染。
- `_invoke_tool()` (`run_agent.py:10902`)：内置工具（`todo`、`session_search`、`memory`、`clarify`、`delegate_task`）优先短路；其余落到 `registry.dispatch`（`tools/registry.py:390-407`）。
- 写入型工具（`memory`）会同步通知 `_memory_manager.on_memory_write()` (`run_agent.py:10957`)，触发外部记忆提供器（Honcho 等）刷新。

### 2.5 三层系统提示词

`_build_system_prompt()` 位于 `run_agent.py:6041-6266`，明确分三层以最大化 prompt cache 命中：

| 层 | 内容 | 缓存策略 |
|----|------|----------|
| **Stable** | `SOUL.md` 或 `DEFAULT_AGENT_IDENTITY`、技能引导（`MEMORY_GUIDANCE`、`SKILLS_GUIDANCE` 等）、工具规范、平台 hint、环境 hint | 长期稳定 |
| **Context** | 调用方提供的 `system_message`、`build_context_files_prompt()` 抓 `AGENTS.md` / `HERMES.md` / `.cursorrules` | 与 CWD 相关 |
| **Volatile** | 内置 `MEMORY.md`/`USER.md` 块、外部记忆 provider 内容、当前时间戳和会话 ID | 每回合可变 |

三层 `\n\n` 拼接后存入 `_cached_system_prompt`。临时 system prompt（`run_conversation` 入参）在 12731-12732 行只用于一次 API 调用，不污染缓存。

### 2.6 上下文压缩

抽象接口 `ContextEngine`（`agent/context_engine.py:32-151`）：

- 字段：`last_prompt_tokens`、`threshold_tokens`、`context_length`、`compression_count`
- 关键方法：`should_compress` / `compress` / `update_from_response` / `update_model`
- 头尾保护：`protect_first_n=3`（系统消息后的前 3 条非系统消息）、`protect_last_n=6`（最近 6 条）
- 阈值默认 75 %（`threshold_percent=0.75`）

内建实现 `ContextCompressor`（`agent/context_compressor.py`）：

1. 在主流程被调用前，先对工具输出做**廉价剪枝**（裁掉过长 tool result）；
2. 调用辅助轻模型（`auxiliary_client.py`）对中间段做摘要；
3. 头尾保留，组合返回新 message list；
4. 触发 SQLite 中 `session_id` 滚动（父子链路通过 `parent_session_id` 维系）。

### 2.7 重试与故障切换

`agent/error_classifier.py` 把 OpenAI / Anthropic / 各 Provider 抛出的异常归类到 `FailoverReason`（rate_limit、auth、quota、context_window、transient、…）。`AIAgent` 在主循环中按类别决定：

- 退避重试（`agent/retry_utils.py:jittered_backoff`）
- 切换凭证（`agent/credential_pool.py`）
- 切换 fallback provider（`hermes_cli/model_switch.py`）
- 触发压缩（`_compress_context`）

### 2.8 回合后异步动作

- **外部记忆同步**：`_sync_external_memory_for_turn` (`run_agent.py:5843`)，向 Honcho 等提供器推送本回合事实；遇到 Ctrl-C 不同步以避免半完整状态污染。
- **记忆 nudge**：累积 `_turns_since_memory ≥ _memory_nudge_interval` 时向 agent 提示考虑写入 MEMORY.md。
- **Curator fork**：累计回合或时间触发后台 review；见 §4.4。

### 2.9 Trajectory 记录

`agent/trajectory.py:30 save_trajectory()`：

- 把转换为 ShareGPT 格式的对话追加到 JSONL（成功 → `trajectory_samples.jsonl`，失败 → `failed_trajectories.jsonl`）。
- `convert_scratchpad_to_think()` 把 `<REASONING_SCRATCHPAD>` 转为 `<think>` 块。
- 剥离 ephemeral system prompt、技能 prefetch、`reasoning_details` 等内部字段，保留模型训练所需信号。

---

## 3. 工具体系

### 3.1 注册与发现：`tools/registry.py`

- `discover_builtin_tools()` (`tools/registry.py:57-74`)：用 AST 解析 `tools/*.py` 检测顶层 `registry.register(...)` 调用，再 import 这些模块。这样**只有真正调用了 register 的模块**才被 import，避免冷启动加载所有 provider SDK。
- `ToolEntry` 数据类 (`tools/registry.py:77-106`)：`name`、`toolset`、`schema`（OpenAI JSON）、`handler`、`check_fn`（运行时可用性探测）、`requires_env`、`is_async`、`emoji`、`max_result_size_chars`、`dynamic_schema_overrides`。
- `registry.register()` (`tools/registry.py:234-305`)：校验 toolset 内无重名（除非 `override=True`），写入 entry，递增 `generation` 计数器供下游缓存失效使用。
- `get_definitions()` (`tools/registry.py:337-384`)：返回当前可用工具 JSON schema 数组，带 30s TTL 缓存（`tools/registry.py:121`）；按 `check_fn` 过滤；应用动态 schema overrides。
- `dispatch()` (`tools/registry.py:390-407`)：执行工具 handler，自动桥接 async（`_run_async`），把异常包成 JSON error string 返回 agent（避免 LLM 看到非字符串结果导致 schema 失败）。

### 3.2 Toolset 概念：`toolsets.py`

- Toolset 是工具的**逻辑分组**，一个工具只属于一个 toolset。
- 支持组合：`toolsets.py:78` 起的 `TOOLSETS` dict 允许某 toolset 引用其他 toolset。
- `_HERMES_CORE_TOOLS` (`toolsets.py:31-73`)：CLI 与所有消息平台共享的核心集合。
- `resolve_toolset(name)`：递归展开为具体工具名集合。
- 平台分发：`toolset_distributions.py` 维护「平台 → 默认 toolset 套餐」映射，例如消息平台默认不带 `moa`/`homeassistant`/`rl` 等高危/重资源 toolset。

`model_tools.py:262-323 get_tool_definitions()` 是聚合入口：合并 `enabled_toolsets - disabled_toolsets`，禁用集会从启用集**减去**，避免组合 toolset 反向激活已禁工具（issue #17309 的处理）。

### 3.3 终端后端

`tools/terminal_tool.py` 的 7 个后端通过 `TERMINAL_ENV` 环境变量选择：

| 后端 | 实现 | 说明 |
|------|------|------|
| `local` | terminal_tool.py 默认 | 直接 host 执行；每条命令前 source session 快照 |
| `docker` | tools/environments/docker.py | 容器隔离 |
| `ssh` | tools/environments/ssh.py | ControlMaster 连接池 |
| `singularity` | tools/environments/singularity.py | Apptainer/Singularity 持久容器 |
| `modal` | tools/environments/modal.py | Modal 云沙箱，serverless 休眠 |
| `daytona` | tools/environments/daytona.py | Daytona IDE 集成 |
| `vercel_sandbox` | tools/environments/vercel.py | Vercel Sandboxes（node24/py3.13） |

启动时探测可用性（Docker daemon、Modal SDK、…），不可用则回退 `local`。

### 3.4 关键内置工具

| 工具 | 文件 | 职责 |
|------|------|------|
| `terminal` | `tools/terminal_tool.py` | shell 执行 + 后台任务 + 中断 + 7 种后端 |
| `browser_*` | `tools/browser_tool.py`、`browser_cdp_tool.py`、`browser_camofox.py` | 基于 a11y tree 的无视觉浏览器自动化；本地 Chromium / Browserbase / Browser Use 三种 backend |
| `read_file` / `write_file` / `patch` / `search_files` | `tools/file_tools.py` | 文件读写 + 设备路径黑名单 + 最大读取字符限制 |
| `execute_code` | `tools/code_execution_tool.py` | 隔离沙盒执行；动态 schema 列出可用工具供子环境调用 |
| `delegate_task` | `tools/delegate_tool.py` | spawn 子 agent；禁用 `delegate_task` / `clarify` / `memory` / `send_message` / `execute_code` 等递归危险工具 |
| `memory` | `tools/memory_tool.py` | 操作 `MEMORY.md` / `USER.md`；写入会冷启动外部 provider |
| `todo` | `tools/todo_tool.py` | 单会话内任务分解栈 |
| `send_message` | `tools/send_message_tool.py` | 跨平台发送消息；按目标名解析路由 |
| `skill_manage` | `tools/skill_manager_tool.py` | 创建 / 编辑 / 打补丁 / 归档 SKILL.md |
| `skills_list` / `skill_view` | `tools/skills_tool.py` | 渐进披露 skill 元数据与正文 |
| `mcp_*` | `tools/mcp_tool.py` | MCP server 桥接 |
| `computer_use` | `tools/computer_use/` | macOS 桌面控制（cua-driver），返回多模态（文本+base64 截图） |

### 3.5 审批流：`tools/approval.py` & `tools/slash_confirm.py`

- 危险命令通过 `DANGEROUS_PATTERNS` 正则识别（rm、shutdown、git push --force …）。
- 状态用 `contextvars.ContextVar` 维持，使网关多线程安全；CLI 回退 `os.environ`。
- 三种交互：
  - CLI：阻塞式 input，回调线程本地。
  - 网关：异步审批队列，按钮 UI（Approve Once / Always / Cancel）或 `/approve` / `/always` / `/cancel` 文字回复。
  - 子代理：默认非交互拒绝；可通过 `delegation.subagent_auto_approve` 配置项 opt-in 自动放行。
- 插件挂钩 `pre_approval_request` / `post_approval_response`（懒导入避免循环）。

### 3.6 MCP 集成：`tools/mcp_tool.py`

- 三种 transport：stdio / HTTP（StreamableHTTP）/ SSE。
- 每个 MCP server 跑在**独立守护线程的 asyncio loop**（`_mcp_loop`），避免阻塞主 agent。
- 工具发现：`tools/list` → 注册到主 registry 的 `mcp-<server_name>` toolset。
- 安全：
  - stdio 子进程仅传 safe env + XDG_* + 用户允许的变量。
  - 工具 description 注入扫描（"ignore previous instructions"、`exec(...)` 等）触发告警。
  - 错误信息按模式剥离凭证。
- Sampling（可选）：MCP server 可反向调用 LLM completion，配置项控制 model / max_tokens / RPM / 工具循环上限。
- 异步桥接 `model_tools._run_async`（`model_tools.py:82-171`）处理三种调用上下文：无主 loop / 已有 loop / worker 线程 → 各对应不同策略，杜绝 "Event loop is closed" 错误。

---

## 4. Skills 系统

### 4.1 Skill 文件格式

```
~/.hermes/skills/<category>/<skill-name>/
├── SKILL.md             # 必需：YAML frontmatter + markdown body
├── references/          # 可选：长篇资料（按需 link）
├── templates/           # 可选：模板文件
├── scripts/             # 可选：辅助脚本
└── assets/              # 可选：二进制资源
```

`SKILL.md` frontmatter 强制字段（`tools/skill_manager_tool.py:217-253 _validate_frontmatter`）：

- `name` ≤ 64 字符、小写连字符
- `description` ≤ 1024 字符、必须以「Use when …」开头
- `version`、`author`、`license`（约定但不强制）
- `metadata.hermes.{tags, related_skills, platforms}`（可选但标准）

整文件长度上限约 100 000 字符（~36k token），peer skill 建议 8-15k。

### 4.2 加载与发现

渐进披露 (Progressive Disclosure)：

| 层级 | API | 数据量 |
|------|-----|--------|
| **Tier 1**（元数据） | `skills_list()` | name + description |
| **Tier 2-3**（正文 + 关联） | `skill_view(name)` | SKILL.md + `references/` / `templates/` / `scripts/` / `assets/` |

启动时**仅装载 Tier 1**，正文按 `/skill-name` 调用时再加载，避免吃光上下文。

`agent/skill_utils.py:92-115` 按 `platforms: [macos, linux, windows]` 字段过滤。`hermes_cli/skills_config.py:27-36` 处理 `skills.disabled` / `skills.platform_disabled` 全局/平台禁用清单。

### 4.3 调用路径：`/skill-name`

1. 用户输入 `/build-mcp-server`（CLI 或网关）
2. `agent/skill_commands.py:53-96 _load_skill_payload()` 规范化标识符 → 调 `skill_view`
3. `_build_skill_message()` 组装注入消息：
   - 模板变量展开（`${HERMES_SKILL_DIR}`、`${HERMES_SESSION_ID}`）
   - 内联 shell 展开（`` !`cmd` `` 语法，可选）
   - 附 `[Skill directory: /abs/path]` 提示，让 agent 知道如何解析 `references/templates/...` 相对路径
   - 注入 `metadata.hermes.config` 中解析后的配置值
4. 消息追加到对话；agent 据此规划工具调用。

### 4.4 "反思阶段" 真相：Curator

README 中的 "Reflective Phase" 在实际代码中**不是回合内学习**，而是 `agent/curator.py:1-250` 实现的**异步后台批合并器**：

- 触发：`maybe_run_curator()`（在 `run_agent.py`）按下列门控：
  - 启用 `curator.enabled`
  - 距上次 ≥ `DEFAULT_INTERVAL_HOURS`（默认 168h / 7 天）
  - agent 当前空闲 ≥ `min_idle_hours`（默认 2h）
- 通过 `_spawn_background_review()` fork 一个**独立 AIAgent** 实例，使用同会话凭证，以 `CURATOR_REVIEW_PROMPT`（`agent/curator.py:330-445`）运行。
- 职责（`agent/curator.py:8-12`）：
  1. 依据使用时间戳推进 `active → stale → archived` 生命周期；
  2. 把窄范围 agent-created skills 合并为「class-level」总技能；
  3. 把过细的会话特定内容迁到 `references/` / `templates/`；
  4. 归档（**永不删除**）真正过期的 skill 到 `.archive/`；
  5. 输出 YAML 报告：consolidations + prunings + 原因。
- 不变量：
  - 仅 touch 使用 sidecar 中 `agent_created=true` 的 skill；
  - 永不 touch 用户固定（`pinned`）或 Hub 安装的 skill；
  - 状态文件 `~/.hermes/skills/.curator_state` 记录 `last_run_at` / `paused` / `run_count`。

### 4.5 Skills Hub：`tools/skills_hub.py`

Hub 把 Skills 当成可分发包：

- `HUB_DIR = ~/.hermes/skills/.hub`
  - `lock.json`：装了哪些 hub 来源的 skill（含 trust_level + content_hash）
  - `quarantine/`：等待安全扫描+用户审批的 skill
  - `audit.log`：安装/移除事件
  - `taps.json`：配置的远端 registry
  - `index-cache/`：远端 index 缓存（TTL 1h）
- `SkillSource` ABC（`tools/skills_hub.py:200+`）：GitHub / clawhub / OpenAI skills / Claude marketplace / LobeHub 五种来源适配器
- `trust_level`：`builtin`（内置）/ `trusted`（openai/anthropics）/ `community`
- `INSTALL_POLICY` 按 trust × scan verdict 决定是否需用户确认

安全扫描在 `tools/skills_guard.py` 实现，50+ 条正则规则覆盖：数据外传、prompt 注入、毁灭性命令、持久化、DNS tunneling……

### 4.6 用量与溯源

`tools/skill_usage.py` 维护 sidecar `~/.hermes/skills/.usage.json`：

- 字段：`state` (active/stale/archived) / `use_count` / `view_count` / `patch_count` / `created_at` / `last_used_at` / `last_viewed_at` / `last_patched_at` / `pinned` / `agent_created`
- 写入：原子（tempfile + os.replace + fcntl/msvcrt 文件锁），失败不抛错。

`tools/skill_provenance.py` 用 `ContextVar` 标记当前写入来源（`foreground` vs `background_review`）。Curator fork 时把 origin 设成 `background_review`，工具据此把新 skill 打上 `agent_created=true`。

### 4.7 Bundled 同步

`tools/skills_sync.py` 把仓库 `/skills/*` 同步到 `~/.hermes/skills/`：

- 新增 → 复制
- 已有 + 用户未改（hash 一致）→ 覆盖更新
- 已有 + 用户改过（hash 不一致）→ 跳过
- 用户删过 → 尊重
- 仓库删过 → 从 `.bundled_manifest` 清理

---

## 5. 消息网关

### 5.1 GatewayRunner

`gateway/run.py:1175` 定义 `GatewayRunner`：

- 装载平台适配器（按 `gateway/platform_registry.py:162-260` 注册表）
- 把入站消息构造成 `SessionSource`（`gateway/session.py:71-89`：平台、chat_id、user_id、thread_id、metadata）
- 调度 `AIAgent` 在线程池中处理
- 通过 `GatewayStreamConsumer`（`gateway/stream_consumer.py:77-150`）把 token 流增量编辑回平台消息

### 5.2 会话与跨平台镜像

- 会话以「平台 + chat_id」键控，持久化到 SQLite (`hermes_state.SessionDB`) + JSONL transcript。
- `mirror.py:25-82 mirror_to_session()`：单独的 delivery-mirror，CLI / cron / 网关均可调用。当往任意平台发消息时，写一条 mirror 记录到目标会话 transcript，让接收侧 agent 拥有「此消息已于 14:05 投递到 Telegram」的语境。

### 5.3 支持的平台（`gateway/platforms/`）

- 即时通讯：Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Mattermost
- 企业即时通讯：飞书 (Feishu)、企业微信 (WeCom)、钉钉、QQ Bot、WeChat
- API / Cloud：API Server、Webhook、Home Assistant
- 其他：Email、SMS、Yuanbao（阿里）、BlueBubbles（iMessage 桥）

平台元数据 `PlatformEntry`：

```
adapter_factory, check_fn, validate_config, is_connected,
standalone_sender_fn,  # 给 cron / 外部进程调用
max_message_length, platform_hint, pii_safe
```

基类 `gateway/platforms/base.py BasePlatformAdapter` 抽象 connect / disconnect / send / receive 生命周期。

### 5.4 配对（DM 授权）：`gateway/pairing.py:76-321`

无静态白名单的代码式授权：

1. 未知用户私聊机器人；
2. 适配器 → `PairingStore.generate_code()` 生成 8 字符无歧义码（去除 0/O、1/I）；
3. 机器人回："请把 `XYZ1234` 发给我以申请访问"；
4. 拥有者 CLI 端 `hermes pair approve telegram XYZ1234` 批准；
5. 用户加入 approved 列表。

安全参数（参 OWASP / NIST SP 800-63-4）：

- `CODE_TTL_SECONDS=3600`、`RATE_LIMIT_SECONDS=600`（用户级）
- `MAX_PENDING_PER_PLATFORM=3`、`MAX_FAILED_ATTEMPTS=5` → `LOCKOUT_SECONDS=3600`
- 文件原子写 + chmod 0600；代码绝不写日志。

### 5.5 定时任务：`cron/`

- 存储：`~/.hermes/cron/jobs.json`，输出到 `~/.hermes/cron/output/<job_id>/<ts>.md`
- 调度：`cron/scheduler.py:9-146 tick()` 60s 心跳，文件锁 `~/.hermes/cron/.tick.lock` 防止并发。
- 投递：`gateway/delivery.py:28-106 DeliveryRouter` 支持四种目标：
  - `origin`：回到 job 创建会话
  - `telegram:<chat_id>` / 其他平台
  - `telegram`（读 `TELEGRAM_HOME_CHANNEL` env）
  - `local`：只落盘
- toolset 门控：cron job 可配 `enabled_toolsets`，或继承 `cron` 平台的默认配置；高危 toolset 默认禁。
- 输出 >4000 字符按平台截断，并附本地完整文件链接。

### 5.6 ACP（Agent Client Protocol）

`acp_adapter/entry.py:237-287` 提供 `hermes-acp` 入口：

- 用 JSON-RPC over stdio 暴露 Hermes 为 ACP server，让 IDE（VS Code、Zed 等）作为 client 接入；
- stdout 严格保留 JSON-RPC，stderr 走 log；
- 服务实现 `acp_adapter/server.py HermesACPAgent`：会话管理 (create/list/load/fork/resume)、工具注册（含 MCP）、资源附件（file URI / blob / embedded）、流式消息 callback。
- 4-worker `ThreadPoolExecutor`（AIAgent 同步执行，用 `asyncio.to_thread` 桥接）。

### 5.7 流式投递

`gateway/stream_consumer.py:113-150`：

- 缓冲 token，按速率（默认 1.5s/次）增量 edit 平台消息；
- 首次 send 后记录 `message_id`，后续 patch；
- "fresh final" 模式：若 preview 在最终前显示已超过 N 秒，最终结果改为发新消息而非 edit。
- `transport="auto"` 优先用平台原生草稿流（Telegram drafts），否则回落 `editMessageText`。

### 5.8 钩子系统

`gateway/hooks.py:35-210`：事件驱动，handler 从 `~/.hermes/hooks/<hook>/handler.py` + `HOOK.yaml` 清单发现。事件：`gateway:startup` / `session:start|end` / `agent:start|step|end` / `command:*`。Handler 失败仅 log，不阻塞主链。

### 5.9 TUI 前端

- `ui-tui/src/` React + Ink；
- 用 `python -m tui_gateway.entry` (`tui_gateway/entry.py:187-250`) 子进程做 JSON-RPC 后端；
- 当指定 `HERMES_TUI_SIDECAR_URL` 时，TUI 还会向 WebSocket 旁路推送事件，供 web dashboard 监控。

---

## 6. CLI、Provider 与状态

### 6.1 CLI 入口与命令分发

- `hermes_cli/main.py:1-44` argparse 顶层调度（`hermes setup` / `hermes model` / `hermes gateway` …）
- 交互式 TUI：`cli.py:7634-8083 process_command()` 处理 slash 命令
- 注册表：`hermes_cli/commands.py:64-350` `COMMAND_REGISTRY`，450+ 命令分 20+ 类别；支持前缀消歧（`/m` → `/model` 仅当唯一）
- 同一注册表被 TUI + 网关共用，保证命令在所有界面一致。

### 6.2 setup 向导

`hermes_cli/setup.py` 分 5 段：

1. Model & Provider（setup.py:789）
2. Terminal Backend（setup.py:1393）：选 shell / SSH 隧道 / PTY 桥接
3. Agent Settings（setup.py:1790）：迭代上限、压缩阈值、reset 策略
4. Messaging Platforms（setup.py:2465）
5. Tools（setup.py:2705）：TTS、图像生成、Web 搜索

每段结束即写 `~/.hermes/config.yaml` + `~/.hermes/.env`，可单独重跑。`--quick` 跳过可选段。

### 6.3 Provider 抽象

`hermes_cli/providers.py:1-150` 两层：

- **Layer 1**：`models.dev` 目录（约 500 模型 / 100+ provider 的 endpoint / env / capability 元数据）
- **Layer 2**：Hermes overlay，定义 transport 类型（`openai_chat` / `anthropic_messages` / `codex_responses`）、auth 模式（`api_key` / `oauth_device_code` / `oauth_external` / `external_process`）、aggregator 标志、base URL 覆盖

支持自定义 provider：用户在 `config.yaml` 写 OpenAI 兼容 endpoint，凭证池里以 `custom:` 前缀键存。

### 6.4 Adapter 模式

每家 Provider 一个 adapter，把 Hermes 内部统一 message 格式翻译到各家 API：

| Adapter | 内容 |
|---------|------|
| `agent/anthropic_adapter.py` (~522 行) | Anthropic Messages API；adaptive thinking 预算（opus-4-7 128k、haiku-4-5 64k）；多种 auth（API key / OAuth / Claude Code credentials） |
| `agent/bedrock_adapter.py` | AWS Bedrock 跨区域故障切换、boto3 session |
| `agent/codex_responses_adapter.py` | OpenAI Codex Responses API（非 chat completions） |
| `agent/gemini_native_adapter.py` + `gemini_cloudcode_adapter.py` | Gemini 多入口 |
| `agent/moonshot_schema.py` | Kimi/Moonshot vision schema quirks |
| `agent/google_code_assist.py` | Google CloudCode |

Adapter 解决：消息格式、工具调用表示、reasoning 块语法、auth header 各家不同。

### 6.5 Model 目录与切换

- `hermes_cli/model_catalog.py:1-70`：远端拉 `hermes-agent.nousresearch.com/docs/api/model-catalog.json`（24h disk cache `~/.hermes/cache/`），网络故障回退硬编码 `_DEFAULT_PROVIDER_MODELS`。
- `hermes_cli/model_switch.py:1-45` 是 CLI `/model` 与网关 `/model` 共用管线：parse flags → 别名解析（claude → claude-opus-4-6） → Provider 解析 → 凭证池查找 → 名称归一化 → 元数据抓取 → 注入 config 或会话本地。
- 支持 fallback chain：主模型不可用时按列表向下试。

### 6.6 凭证池

`agent/credential_pool.py:1-150` + `agent/credential_sources.py:1-110`：

- `PooledCredential` 数据类，持久化到 `~/.hermes/auth.json`
- 来源前缀：`env:<VAR>` / `claude_code` / `hermes_pkce` / `device_code` / `qwen-cli` / `gh_cli` / `config:<name>` / `manual`
- 选择策略：`fill_first` / `round_robin` / `random` / `least_used`
- 状态：`last_status`、`last_error_code`、`error_reset_at`（401 冷却 5min，429/402 冷却 1h）
- 移除：统一 `RemovalStep` 注册表，`hermes auth remove` 真正擦除来源（.env 行、auth.json 块、OAuth 文件），不会下次启动又被加载。

### 6.7 OpenClaw 迁移

`hermes_cli/claw.py:1-54` 调用 `optional-skills/migration/openclaw-migration`：

- 探测 `~/.openclaw` 与 OpenClaw 进程（systemd / tasklist / PowerShell）
- 迁移内容：`SOUL.md`、`MEMORY.md` / `USER.md`、用户自建 skills → `~/.hermes/skills/openclaw-imports/`、命令审批 allowlist、消息平台配置、API keys（Telegram / OpenRouter / OpenAI / Anthropic / ElevenLabs）、TTS 资源、工作区指令 (`AGENTS.md`)
- 预迁移快照存盘以便回滚（claw.py:90-100）
- 预设模式：`--dry-run` / `--preset user-data`（不含 secrets）/ `--overwrite`

### 6.8 SessionDB（SQLite，schema v11）

`hermes_state.py:185-276` 表结构 + `309+` `SessionDB` 类：

`sessions` 表（部分字段）：
- `id`、`source`、`user_id`、`model`、`model_config`
- `started_at` / `ended_at` / `end_reason`
- `message_count` / `tool_call_count`
- `input_tokens` / `output_tokens` / `cache_read_tokens` / `cache_write_tokens` / `reasoning_tokens`
- `estimated_cost_usd` / `actual_cost_usd`
- `title` / `parent_session_id`（压缩触发的会话父子链）
- `handoff_state` / `handoff_platform` / `handoff_error`（跨平台交接）

`messages` 表（部分字段）：
- `id`、`session_id`、`role`、`content`、`tool_call_id`、`tool_calls`、`tool_name`
- `reasoning` / `reasoning_content` / `reasoning_details`（Anthropic adaptive thinking）
- `codex_reasoning_items` / `codex_message_items`（Codex Responses API 专属）

`messages_fts` 虚表（FTS5，state.py:253-276）：对 `content` + `tool_name` + `tool_calls` 建全文索引，给 `session_search` 工具 + `/insights` 等命令用。

WAL 模式默认；探测到 NFS/SMB/FUSE 无法加锁时回退 DELETE journal（state.py:128-183），保证云盘/共享盘也能跑。

---

## 7. 关键文件 / 行号速查表

| 主题 | 入口 |
|------|------|
| 主循环 | `run_agent.py:12526-15285` |
| 工具调度 | `run_agent.py:10860, 11014, 11422, 10902` |
| 系统提示词三层 | `run_agent.py:6041-6266`（接 `agent/prompt_builder.py`） |
| 压缩 | `run_agent.py:10641, agent/context_compressor.py` |
| 记忆 | `run_agent.py:1978-2000, 5818-5890, 6212-6230, 10944-10970` |
| Curator | `agent/curator.py:1-250` |
| Trajectory | `agent/trajectory.py:1-57` |
| Tool registry | `tools/registry.py:151, 234, 337, 390` |
| Toolset | `toolsets.py:78, model_tools.py:262` |
| MCP | `tools/mcp_tool.py:95-381, model_tools.py:82-171` |
| 审批 | `tools/approval.py, tools/slash_confirm.py:51-80` |
| Gateway | `gateway/run.py:1175, session.py:71, mirror.py:25, stream_consumer.py:77` |
| Pairing | `gateway/pairing.py:76-321` |
| Cron | `cron/scheduler.py:9-146, gateway/delivery.py:28-106` |
| ACP | `acp_adapter/entry.py:237-287, acp_adapter/server.py:1-150` |
| Skill 校验 | `tools/skill_manager_tool.py:217-253, 373-427` |
| Skill 加载 | `agent/skill_commands.py:53-96, agent/skill_utils.py:92-115` |
| Skills Hub | `tools/skills_hub.py:200+, tools/skills_guard.py` |
| 用量 / 溯源 | `tools/skill_usage.py, tools/skill_provenance.py` |
| CLI | `cli.py:7634-8083, hermes_cli/commands.py:64-350, hermes_cli/main.py` |
| Setup | `hermes_cli/setup.py:789-2700` |
| Provider | `hermes_cli/providers.py:1-150` |
| Adapter | `agent/anthropic_adapter.py, codex_responses_adapter.py, gemini_native_adapter.py, gemini_cloudcode_adapter.py, bedrock_adapter.py` |
| Model 切换 | `hermes_cli/model_switch.py, model_catalog.py` |
| 凭证池 | `agent/credential_pool.py, credential_sources.py` |
| OpenClaw 迁移 | `hermes_cli/claw.py:1-100, optional-skills/migration/openclaw-migration` |
| SessionDB | `hermes_state.py:185-276, 309+` |
