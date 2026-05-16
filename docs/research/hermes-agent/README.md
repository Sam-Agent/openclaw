# Hermes Agent 研究档案

> 对 [Nous Research / hermes-agent](https://github.com/NousResearch/hermes-agent) 项目的代码分析与技术架构研究。
>
> 研究分支：`claude/hermes-agent-research-ffwrb`
> 研究对象版本：v0.13.0（commit 抓取于 2026-05-16）
> 上游许可证：MIT
> 上游仓库规模：约 105 MB，~81.6 万行 Python

## 文档清单

| 文档 | 内容 |
|------|------|
| [code-analysis.md](./code-analysis.md) | 代码分析：源码组织、关键模块、核心类与函数的实现细节，附文件:行号引用 |
| [technical-architecture.md](./technical-architecture.md) | 技术架构：分层模型、运行时数据流、子系统协作关系、扩展点 |
| [architecture.svg](./architecture.svg) | 架构总览图：六层结构（Interaction / Orchestration / Provider / Capability / Execution / Persistence） |
| [dataflow.svg](./dataflow.svg) | 数据流图：CLI 单回合 + 网关单回合 + cron 定时任务三条主路径 |

## 研究方法

1. 在临时目录 `/tmp/hermes-research/hermes-agent` 执行 `git clone --depth 1` 浅克隆上游仓库；
2. 阅读 `README.md`、`pyproject.toml`、`AGENTS.md` 建立项目全貌；
3. 并行派发五个 Explore 子代理分别深入下列子系统：
   - Agent 主循环（`run_agent.py` / `agent/` 包）
   - 工具体系（`tools/` / `model_tools.py` / `toolsets.py`）
   - Skills 系统（`skills/` / `tools/skill_manager_tool.py` / `agent/curator.py`）
   - 消息网关（`gateway/` / `cron/` / `acp_adapter/`）
   - CLI 与 Provider 抽象（`cli.py` / `hermes_cli/` / `providers/` / `agent/*_adapter.py`）
4. 汇总各子代理的输出，结合一手源码引用，写出本目录下两份分析文档。

## 与 OpenClaw 的关系（背景）

Hermes Agent 是 Nous Research 自 OpenClaw 衍生（或对标）的自托管 AI Agent，README 中含有从 OpenClaw 迁移的明确路径（`hermes claw migrate`），并在 `hermes_cli/claw.py` 中实现了 `~/.openclaw → ~/.hermes` 的整套数据迁移。本研究面向 OpenClaw 工程团队，便于参照其架构选型。
