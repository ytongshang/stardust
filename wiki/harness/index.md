---
title: Agent Harness
type: concept-index
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent
  - uxa-center
tags:
  - agent-harness
  - runtime
  - uxarts-agent
---

# Agent Harness

这个目录记录 agent harness 的概念树。重点不是每天收集项目，而是把 agent 运行时拆成稳定小类：每类先说明 `uxarts-agent` 现在是什么、做什么、怎么做；后续看到 GitHub 项目、X/Twitter thread、文章、论文时，只补充真正新的实现思路。

## 使用原则

- 不以“项目”为中心，以“概念”为中心。
- 外部来源可以是 GitHub、X/Twitter、文章、论文、release notes。
- 如果外部实现只是换名覆盖已有概念，只在对应文件的 `Related Implementations` 里记一行。
- 如果外部实现有新的边界、失败恢复、eval、policy、trace 或上下文策略，再单独展开。
- 业务词和 runtime 词分开：`mode/stage/sub_agent_type` 是业务路由；input/context/tool/workspace/lifecycle 才是通用 harness 概念。

## 概念树

### Runtime

- [[harness/runtime/agent-loop|Agent Loop]] — 模型调用、工具执行、handoff、transfer、interrupt、final output。
- [[harness/runtime/run-ledger|Run Ledger]] — `RunItem` 事件流，如何替代普通 messages 成为可恢复事实日志。

### Context

- [[harness/context/input-transform|Input Transform]] — raw input 到 LLM user message 的转换。
- [[harness/context/context-injection|Context Injection]] — 首轮、每轮、压缩后、子 agent 启动时注入什么。
- [[harness/context/compaction|Compaction]] — 触发、summary、`UxMessageTrimItem`、压缩后上下文拼接。

### Tools / Policy

- [[harness/tools/tool-system|Tool System]] — tool schema、参数修复、错误回传、并行/顺序执行、interrupted tool。
- [[harness/tools/tool-profile-policy|Tool Profile & Policy]] — stage/mode 工具裁剪、plan mode、sub-agent 权限、plugin/skill 动态工具。

### Workspace / Memory

- [[harness/workspace/workspace-artifacts|Workspace & Artifacts]] — `Codebase`、attachment、snapshot、editor policy、sub-agent merge。
- [[harness/memory/memory-capability|Memory & Capability]] — project memory、wiki memory、plugin/skill/capability snapshot。

### Multi-Agent

- [[harness/multi-agent/subagents|Subagents]] — inline subagent、background subagent、AskMainAgent、SendMessage、TaskStop、merge。

### Lifecycle

- [[harness/lifecycle/center-orchestrator|Center Orchestrator]] — queue、heartbeat、pending、retry、complete、stop。

### Observability / Research

- [[harness/observability/trace-eval|Trace & Eval]] — run item trace、hooks、fake item report、未来 replay/eval。
- [[harness/research/collection-workflow|Research Collection Workflow]] — 如何收集 GitHub/X/文章/论文，并按概念去重。

## uxarts-agent 主要代码入口

- `uxarts-agent/uxarts/core/agent/agent_runner.py`
- `uxarts-agent/uxarts/core/agent/types/items.py`
- `uxarts-agent/uxarts/core/agent/types/tool.py`
- `uxarts-agent/uxarts/core/agent/tools/tool_runner.py`
- `uxarts-agent/uxarts/web/modules/getui/task/getui_generate_task.py`
- `uxarts-agent/uxarts/web/modules/getui/task/getui_runner.py`
- `uxarts-agent/uxarts/web/modules/getui/agent/types/getui_context.py`
- `uxarts-agent/uxarts/web/modules/getui/task/conversation_context_window.py`
- `uxarts-agent/uxarts/web/modules/getui/task/async_sub_agent/sub_agent_run_registry.py`
- `uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentApplicationCommandService.java`
- `uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus/AgentCompleteMessageListener.java`

## Open Questions

- `RuntimeProfile` 应先只读落地，还是先从 `SubAgentRunSpec` 横向推广？
- `RunItem + RuntimeProfile` 是否足够支撑 trace replay？
- Tool policy 和 workspace policy 是否应统一成一个 permission layer？
- 压缩 summary 的质量应该如何自动评估？
