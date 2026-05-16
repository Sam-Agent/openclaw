# Hermes Agent 技术架构

> 与 [code-analysis.md](./code-analysis.md) 配套。代码分析关注**实现细节**，本文档关注**分层、数据流与扩展点**。

## 1. 设计哲学（自 README + 源码诠释）

| 主张 | 落地方式 |
|------|----------|
| **Self-improving** | 后台 Curator 每隔默认 7 天扫描 `agent_created` 类 skill，做合并/归档，从经验里形成「class-level」总技能 |
| **Lives where you do** | 单一 `GatewayRunner` 进程同时驱动 Telegram / Discord / Slack / WhatsApp / Signal / 飞书 / 钉钉 / Email…，会话级镜像保证跨平台连续性 |
| **Use any model** | Provider × Adapter 双层抽象 + 凭证池 + 模型目录 + fallback chain；切换不改代码 |
| **Run anywhere** | 终端工具 7 后端（local / docker / ssh / singularity / modal / daytona / vercel_sandbox），后两者支持 serverless 休眠 |
| **Research-ready** | 单个 trajectory.py 把每次回合保存为 ShareGPT 格式 JSONL，供训练下一代工具调用模型 |
| **Supply-chain safety** | 所有直接依赖精确钉版 + `uv.lock`；optional extras 懒装；MCP 子进程 env 白名单 + tool description 注入扫描 |

## 2. 高阶分层

```
┌─────────────────────────────────────────────────────────────────────┐
│  Interaction Layer                                                  │
│  ├── CLI / TUI   (cli.py + ui-tui React/Ink + tui_gateway JSON-RPC) │
│  ├── ACP server  (acp_adapter/ → IDE 集成)                          │
│  └── Gateway     (gateway/run.py → 多平台适配器)                    │
└─────────────────────────┬───────────────────────────────────────────┘
                          │ 入消息 / 出 token 流
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Orchestration Layer                                                │
│  ├── AIAgent (run_agent.py)         ← 主回合循环                    │
│  ├── Context Engine                 ← 触发压缩                      │
│  ├── Iteration Budget + Interrupts  ← 资源 / Ctrl-C 控制            │
│  └── Trajectory writer              ← 训练数据落盘                  │
└─────────────────────────┬───────────────────────────────────────────┘
                          │ chat.completions / messages / responses
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Provider Layer                                                     │
│  ├── Provider DB (models.dev + Hermes overlay)                      │
│  ├── Per-provider Adapters (anthropic / codex / gemini / bedrock …) │
│  ├── Credential Pool + Sources                                       │
│  └── Failover chain                                                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Capability Layer (按需挂载)                                        │
│  ├── Tools (registry.py + 50+ 内置 + MCP 桥接)                      │
│  ├── Skills (procedural memory; SKILL.md 渐进披露)                  │
│  ├── Memory (MEMORY.md + USER.md + 外部 provider Honcho)            │
│  ├── Cron scheduler                                                  │
│  └── Subagents (delegate_task)                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Execution Layer                                                    │
│  ├── Terminal backends (local / docker / ssh / singularity /        │
│  │                       modal / daytona / vercel_sandbox)          │
│  └── Browser backends (Local Chromium / Browserbase / Browser Use)  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Persistence Layer                                                  │
│  ├── SessionDB  SQLite + FTS5 (~/.hermes/sessions.db)               │
│  ├── ~/.hermes/skills/ (bundled + user + .hub/)                     │
│  ├── ~/.hermes/credentials/, auth.json, .env                        │
│  ├── ~/.hermes/cron/                                                │
│  └── ~/.hermes/pairing/                                             │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. 运行时数据流

### 3.1 单回合（CLI）

```
User Input ──▶ cli.process_command()
   ├─ 是 /xxx slash？ ── 命令注册表分发 ── 回写结果到 TUI
   └─ 普通消息
       └─▶ AIAgent.run_conversation()
             ├─ 1. 拼装/复用三层 system prompt（含 SOUL/MEMORY/SKILLS/env）
             ├─ 2. preflight 估 token → 如超阈值，ContextCompressor.compress()
             ├─ 3. 选 provider + credential（pool 策略）
             ├─ 4. adapter 转换 → 调 LLM（streaming 优选 + worker 线程包裹）
             ├─ 5. error classifier 分类
             │       ├─ retry / backoff
             │       ├─ rotate credential
             │       ├─ context-window → 触发 compress 重试
             │       └─ fallback provider
             ├─ 6. 解析 response → tool_calls?
             │       ├─ 有：_execute_tool_calls (并发或串行)
             │       │     ├─ memory 写 → 通知外部 memory provider
             │       │     ├─ skill_manage → SKILL.md atomic write + provenance
             │       │     ├─ terminal → 选定 backend 执行 + 审批门控
             │       │     ├─ mcp_* → bridge 到独立 asyncio loop
             │       │     └─ delegate_task → spawn 子 AIAgent（toolset 缩水）
             │       │   → tool result 追加为 tool message → 继续循环
             │       └─ 无：发出最终文本 → 流式回写 TUI
             ├─ 7. 回合后：trajectory.save_trajectory() + memory_nudge 检查
             └─ 8. 满足门控 → spawn 后台 Curator fork（独立 AIAgent）
```

### 3.2 单回合（消息网关）

```
Telegram update / Slack event / Discord message
   └─▶ PlatformAdapter.on_message()
         ├─ user 不在 approved？ → PairingStore.generate_code()，回复指引
         └─▶ GatewayRunner.dispatch()
               ├─ SessionSource 构造 + Session 加载（SQLite + transcript JSONL）
               ├─ 把 fired hooks: session:start → handler
               ├─ enqueue 到线程池 → AIAgent.run_conversation()
               │     ├─ stream_delta_callback → GatewayStreamConsumer
               │     │     └─ 按 1.5s 节流 editMessage / send draft
               │     ├─ 工具调用同 §3.1
               │     │     └─ send_message tool → 跨平台投递（DeliveryRouter）
               │     │           └─ mirror_to_session 写一条 mirror 记录到目标会话
               │     └─ 危险命令 → 异步审批队列（按钮 UI / /approve 文本回退）
               └─ 完成 → fire hook session:end + 持久化 token/cost 统计
```

### 3.3 定时任务

```
cron/scheduler.tick()  (60s 心跳, 文件锁防并发)
   └─ 扫 ~/.hermes/cron/jobs.json
       └─▶ 到期 job → ThreadPool spawn AIAgent (按 job 自带 enabled_toolsets)
             ├─ AIAgent 执行（无 user，直接拼 system prompt + 任务 prompt）
             └─ 输出 → DeliveryRouter
                   ├─ "origin" → 写回创建会话 transcript
                   ├─ "telegram:<chat>" / "discord:..." → 平台 send
                   ├─ "telegram" (home) → 读 TELEGRAM_HOME_CHANNEL env
                   └─ "local" → 落到 ~/.hermes/cron/output/
```

## 4. 关键架构选择

### 4.1 三层 System Prompt + 单回合复用

`_cached_system_prompt` 一会话只构造一次（首回合），后续直接复用。原因：

- **保住 prefix cache**：Anthropic/OpenAI 长 prefix 缓存对成本与 TTFT 影响巨大；prompt 任何字节变动都会击穿缓存。
- **三层结构**让真正会变的（时间戳、记忆增量）位于尾部 volatile 层，重写它不影响 stable+context 层的 prefix 命中。
- 临时 ephemeral prompt 通过 `run_conversation` 参数注入，仅用于单次调用，不污染缓存。

### 4.2 Context Engine 抽象 + 默认压缩器

`agent/context_engine.py` 把压缩做成可插拔接口（默认 `ContextCompressor`，亦支持 LCM 等第三方）。配置项 `context.engine`。这样：

- 第三方研究可以在不改主循环的前提下实验 DAG / LCM / retrieval-based 上下文管理；
- 引擎可自带工具（如 `lcm_grep`）— `get_tool_schemas()` + `handle_tool_call()`；
- token 状态由引擎自维护，主循环只读 `last_prompt_tokens` / `threshold_tokens` 等公共字段。

### 4.3 工具发现的 AST 预扫

`tools/registry.discover_builtin_tools()` 用 AST 解析判定哪些模块**真的注册了工具**才 import 它们。避免冷启动加载所有 provider SDK（其中很多只在某些工具被启用时才需要）。配合 `lazy_deps.py` 的按需安装，启动开销可控。

### 4.4 Toolset 分组 + 平台分发

- 工具属于唯一 toolset（如 `web`、`browser`、`terminal`）；
- toolset 可组合；
- `toolset_distributions.py` 维护「平台 → 默认 toolset 套餐」：消息平台**默认禁用** `moa`、`homeassistant`、`rl`，避免远程触发高危/重资源能力；
- 用户态用 `hermes tools` 微调。

### 4.5 凭证池 = 多账号 + 多来源 + 策略

`auth.json` 把同一 provider 的多条凭证组织成 pool，分别记 `last_status` / `last_error_code` / `error_reset_at`。意义：

- 个人多家 OpenRouter / Claude API 账号自动轮换；
- 401（auth）冷却 5min、429/402（rate/quota）冷却 1h，避免对失效 key 反复打；
- 来源插件化（env / claude_code / hermes_pkce / device_code / qwen-cli / gh_cli / config / manual），新增 OAuth flow 只新增一种来源即可。

### 4.6 Per-Provider Adapter

Provider 不能用一套通用 `chat.completions` 抹平：

- Anthropic Messages：content blocks、thinking blocks、cache_control 标记位置不同；
- Codex Responses：function_call_id 派生算法专属（`_codex_derive_responses_function_call_id`）；
- Gemini protobuf：schema 字段命名不同；
- Bedrock：跨区域 retry 与 boto3 session；
- 旧 LMStudio：reasoning 字段格式与 OpenAI 不一致。

所以 `agent/*_adapter.py` 直接以模型族为单位写专门转换函数 + `_codex_split_responses_tool_id` 等 helper。

### 4.7 子代理隔离（`delegate_task`）

`tools/delegate_tool.py` 让 agent 自己 spawn agent：

- 子 agent 用同 provider/model；
- toolset 缩水：移除 `delegate_task`（防递归爆炸）、`memory` / `send_message`（防偏离任务写入主用户记忆）、`clarify` / `execute_code`（子环境上下文不足）；
- 子代理结果以 tool result 形式回填主 agent；
- 上下文成本：主 agent 只看到 delegate_task 的输入输出，子代理多步消耗与压缩在子 session 内自治。

### 4.8 Skills = Procedural Memory（不是知识库）

Skill ≠ RAG 文档：

- **格式约束**：description 必须以 "Use when …" 起头，强制可路由；
- **渐进披露**：仅 description 进 system prompt（每条几十 token），正文按需加载，绕开「全部塞进 prompt」反模式；
- **生命周期**：active → stale → archived（**永不删**）；
- **来源溯源**：foreground 用户驱动 vs background_review curator 创建；
- **Curator 不变量**：只 touch agent_created skill，永不动用户固定 / Hub 装的，**只归档不删**。

这套设计让 README 标榜的「self-improving」可信：增量是真正的小步、可撤回的合并/归档，而非把对话内容打包进黑盒。

### 4.9 SQLite + FTS5 + WAL fallback

`hermes_state.SessionDB`：

- WAL 让 gateway 多线程读 + 主线程写并发；
- 探测 NFS/SMB/FUSE 时回退 DELETE journal（因为 WAL 依赖文件锁）；
- FTS5 虚表给 `session_search` 工具用，跨会话回忆走 SQLite 而非长 prompt；
- 父子 session 链 `parent_session_id` 把压缩前后串成 DAG。

### 4.10 ACP 让 IDE 把 Hermes 当一等 Agent 用

把 Hermes 包成 ACP server（JSON-RPC over stdio）后，VS Code / Zed 等 ACP client 可以：

- 创建/分叉/恢复会话；
- 注册 MCP 工具；
- 双向流送消息/工具进度；
- 设置 reasoning mode / model。

这让 Hermes 同时是「自托管 chatbot」**和**「IDE agent」，对照其他单形态项目是个清晰差异点。

## 5. 扩展点（Plugin Surface）

| 扩展类别 | 入口 | 典型用途 |
|----------|------|----------|
| Context Engine | `plugins/context_engine/<name>/` + `register()` | 替换默认压缩器（如 LCM） |
| Platform Adapter | `gateway/platforms/<name>/` + `platform_registry.register()` | 接入新 IM |
| Tool | `tools/<name>.py` + `registry.register()` | 任何新能力 |
| MCP server | `~/.hermes/config.yaml mcp_servers:` | 用现成 MCP 生态接入 |
| Skill | 写一个 SKILL.md 到 `~/.hermes/skills/` 或上传到 Hub | 程序化记忆 |
| Hook | `~/.hermes/hooks/<event>/handler.py` + `HOOK.yaml` | 网关事件钩子 |
| Provider | `providers/` + `models.dev` 条目 + adapter | 新 LLM 厂家 |
| Credential Source | `agent/credential_sources.py` 注册新 prefix | 新 OAuth flow |
| Terminal Backend | `tools/environments/<name>.py` | 新执行沙箱 |
| Cron Delivery Target | `gateway/delivery.py DeliveryRouter` | 新投递路径 |

## 6. 跨子系统不变量

- **会话 ID 是唯一的「身份」**：跨 SQLite、JSONL、cron 输出、curator state、provenance 都用同一 ID。
- **Session_id 在压缩时滚动**，新 session 通过 `parent_session_id` 指向旧 session；trajectory 因此能复原 DAG。
- **审批状态是 ContextVar**：保证 gateway 多线程下不串号；CLI 回落 env 变量。
- **凭证文件 chmod 0600**、配对码不写日志、MCP env 白名单：安全姿态贯穿到具体实现。
- **凡是会被多进程/多线程写的状态**（usage sidecar、cron jobs、pairing store、curator state）都走「tempfile + os.replace + fcntl/msvcrt 文件锁」原子写。
- **永不删除**原则反复出现：skill 归档而非删、auth 移除有专门 RemovalStep、cron 输出本地完整保留。

## 7. 与 OpenClaw 的可比性

| 维度 | OpenClaw | Hermes |
|------|----------|--------|
| 主要语言 | TypeScript (Node 22+) | Python 3.11+ |
| 入口 | `pnpm openclaw` / `pnpm dev` | `hermes` / `hermes-agent` |
| 状态存储 | 默认 JSON + Pi 会话 JSONL | SQLite v11 + FTS5 + WAL |
| Channels | TS 内核 + `extensions/*` plugin（Telegram、Discord、Slack、Signal、iMessage、Web、MS Teams、Matrix、Zalo、Voice） | `gateway/platforms/` + 注册表（增加飞书、企微、钉钉、Yuanbao） |
| Skills | 受 `agentskills.io` 约束（兼容） | 同约束 + 额外 Curator 后台合并 + Hub trust level |
| 终端后端 | 单一（本地）+ 可扩展 | 7 种 + serverless（modal、daytona、vercel） |
| Model 抽象 | provider-web + 各 Provider 客户端 | 100+ provider，per-provider adapter |
| IDE 集成 | docs.acp 协议存在但未作为顶层 | `hermes-acp` 命令直接暴露 |
| 上下文压缩 | 单一压缩器 | 抽象 `ContextEngine` 可换 |

迁移面（`hermes_cli/claw.py`）已经把两侧的差异收敛到一套数据导入路径：SOUL.md、MEMORY.md、用户 skills、API keys、消息平台配置等都能从 `~/.openclaw` 拷过去；这意味着 Hermes 把 OpenClaw 当成「能直接接管的同代竞品」对待，而不是另一个生态。

## 8. 风险与权衡观察

1. **`run_agent.py` 单文件 16k 行** — 集中度极高，重构成本巨大；与官方 `agent/` 包提取的努力（curator、prompt_builder、context_compressor 已外迁）相反。新人很难一次掌握主循环。
2. **Curator 异步 fork** — 用同 provider/model 做后台合并 → 长期使用会持续产生**第二份 API 费用**；该费用对最终用户不直观。
3. **依赖精确钉版** — 安全上正确，但意味着任何 CVE 都必须由维护者主动 bump + 重发；对小团队是负担。这也是 `pyproject.toml` 注释里详细解释 2026-05 Mini Shai-Hulud 蠕虫教训的原因。
4. **AST 工具发现** — 通过解析模块顶层 `registry.register()` 决定 import，能省冷启动，但对装饰器式注册或动态注册不友好。新增贡献者写工具时容易踩坑。
5. **三层 prompt 缓存策略要求 Adapter 协作** — 每个 adapter 必须正确放 `cache_control` 标记；任何疏忽都直接把 cache 命中打掉。Per-provider adapter 由此承担额外验证职责。
6. **Skills Hub 信任模型** — 三档 trust_level 看起来谨慎，但 `safe` verdict + `community` 来源仍直接允许装载；50+ 正则覆盖广但难穷尽 prompt-injection 演变。
7. **多平台审批靠 ContextVar** — 线程模型不可改，未来要支持 async-only 网关（如 aiohttp）需要再次抽象。

---

## 9. 一句话总结

Hermes Agent 把「**单体 AIAgent 类 + 抽象 ContextEngine + Toolset/Skill 双层能力 + Provider/Adapter/CredentialPool 三件套 + 多平台 Gateway**」组合成一个自托管、可移植、可被 IDE 当一等 Agent 嵌入的 Python 进程；并通过 **后台 Curator + Skills Hub + SessionDB FTS5** 把「记忆与经验」沉淀为可审计、可撤回、可追溯的程序化对象。
