---
title: Agent Harness
type: concept-index
created: 2026-06-20
updated: 2026-06-21
sources:
  - uxarts-agent
  - uxa-center
tags:
  - agent-harness
  - runtime
---

# Agent Harness

这个目录记录 agent harness 的概念树。一级目录是大域，二级目录是可以横向比较的通用 harness 概念。每个概念目录下的 `index.md` 写概念本身，`implementations/` 放不同项目的实现笔记；`uxarts-agent` 只是第一个 implementation。

## 收录规则

- 一级目录放大域，不放项目名。
- 二级目录必须是通用问题：外部 harness 也可能有对应实现，即使名字不同。
- uxarts-agent 特有的 `mode/stage/round_extra/CapabilitySnapshot` 等，只写进对应 `implementations/uxarts-agent.md`。
- 外部材料如果只是覆盖已有概念，只在对应目录下简短记录；有新的抽象、失败恢复、eval、policy、trace、上下文策略时再单独展开。
- Research 是发现与分流层，不算 harness 本体概念。
- 每日 radar 的执行流程、源码下载和去重状态放在
  `/Users/rancune/Work/agent/automations/agent-harness-radar/`，不要写进 wiki。
- 每日 radar 先写到 stardust 分支 `radar/YYYY-MM-DD`，次日复核后再决定是否合入概念树。

## 概念树

### Runtime

- [[harness/runtime/agent-loop/index|Agent Loop]] — 模型调用循环、工具调用解析、继续/结束判断。
- [[harness/runtime/event-log/index|Event Log]] — 可恢复的运行事件流，uxarts-agent 里对应 `RunItem`。
- [[harness/runtime/handoff-transfer/index|Handoff & Transfer]] — agent 间移交、流程转交与当前执行主体切换。
- [[harness/runtime/runtime-brokerage/index|Runtime Brokerage]] — 多个 external runtimes 的路由、resume、fallback 和能力归一化。

### Context

- [[harness/context/input-transform/index|Input Transform]] — raw user/event input 到 LLM messages。
- [[harness/context/prompt-assembly/index|Prompt Assembly]] — system prompt、workflow、rules、tool instructions 的组装。
- [[harness/context/context-injection/index|Context Injection]] — 首轮、每轮、压缩后等注入时机。
- [[harness/context/context-budgeting/index|Context Budgeting]] — token 预算、图片预算、缓存友好上下文组织。
- [[harness/context/compaction/index|Compaction]] — 历史压缩、summary、trim、恢复拼接。

### Tooling

- [[harness/tooling/tool-definition/index|Tool Definition]] — tool schema、函数签名、参数模型。
- [[harness/tooling/tool-execution/index|Tool Execution]] — tool runner、并行/顺序执行、执行上下文。
- [[harness/tooling/tool-results/index|Tool Results]] — tool observation/result envelope、messages、attachments。
- [[harness/tooling/tool-error-recovery/index|Tool Error Recovery]] — 参数修复、unknown tool、错误转可恢复结果。
- [[harness/tooling/tool-interruption/index|Tool Interruption]] — 工具挂起、人机协作、等待外部输入。
- [[harness/tooling/tool-availability/index|Tool Availability]] — 动态暴露、裁剪、路由工具集合。
- [[harness/tooling/tool-permissions/index|Tool Permissions]] — 工具级权限、路径限制、main/sub-agent 边界。
- [[harness/tooling/remote-tools/index|Remote Tools]] — MCP/plugin manifest/hosted tools 等远程工具。
- [[harness/tooling/extension-registry/index|Extension Registry]] — plugin/capability/catalog/manifest 等扩展注册。

### Skills

- [[harness/skills/skill-discovery/index|Skill Discovery]] — 如何让模型发现可用 skill。
- [[harness/skills/skill-loading/index|Skill Loading]] — 何时把 skill 内容加载进上下文。
- [[harness/skills/skill-materialization/index|Skill Materialization]] — skill 作为文件包/资源落地。
- [[harness/skills/skill-versioning/index|Skill Versioning]] — skill 版本、更新、兼容性。
- [[harness/skills/skill-evaluation/index|Skill Evaluation]] — 衡量 skill 是否真的改变 agent 行为。

### Workspace

- [[harness/workspace/workspace-model/index|Workspace Model]] — 工作区抽象、文件系统/VFS/product artifact 边界。
- [[harness/workspace/artifacts/index|Artifacts]] — 产物读写、附件、发布、删除。
- [[harness/workspace/snapshots/index|Snapshots]] — 快照、tombstone、历史视图。
- [[harness/workspace/sandbox/index|Sandbox]] — 隔离执行、run_code、预览、测试。
- [[harness/workspace/workspace-merge/index|Workspace Merge]] — 子工作区回主工作区、冲突合并。

### Memory

- [[harness/memory/working-memory/index|Working Memory]] — 当前任务/会话内短期状态。
- [[harness/memory/long-term-memory/index|Long-Term Memory]] — 跨 session 的持久知识。
- [[harness/memory/memory-retrieval/index|Memory Retrieval]] — 记忆召回与选择。
- [[harness/memory/memory-persistence/index|Memory Persistence]] — 记忆保存、同步、清理。

### Multi-Agent

- [[harness/multi-agent/subagents/index|Subagents]] — 子 agent 基础抽象与委托。
- [[harness/multi-agent/background-agents/index|Background Agents]] — 独立异步 agent session。
- [[harness/multi-agent/agent-communication/index|Agent Communication]] — agent 间消息、询问、回投。
- [[harness/multi-agent/teams/index|Teams]] — 多 agent team/role/组织方式。
- [[harness/multi-agent/coordination/index|Coordination]] — fanout、任务归属、依赖、结果协调。

### Lifecycle

- [[harness/lifecycle/queueing/index|Queueing]] — 全局/会话/轮次队列。
- [[harness/lifecycle/heartbeat-retry/index|Heartbeat & Retry]] — lease、心跳、重试。
- [[harness/lifecycle/pending-resume/index|Pending & Resume]] — 挂起、外部回复、续跑。
- [[harness/lifecycle/completion/index|Completion]] — complete、终态、完成事件。
- [[harness/lifecycle/cancellation/index|Cancellation]] — stop/cancel/soft stop。
- [[harness/lifecycle/durable-execution/index|Durable Execution]] — 长任务耐久性与恢复。

### Guardrails

- [[harness/guardrails/permissions/index|Permissions]] — 最小权限、路径/工具权限。
- [[harness/guardrails/approvals-hitl/index|Approvals & HITL]] — 人工确认、人机协作。
- [[harness/guardrails/sandbox-policy/index|Sandbox Policy]] — 沙箱边界和执行策略。
- [[harness/guardrails/risk-control/index|Risk Control]] — content / security / application risk control。

### Observability

- [[harness/observability/tracing/index|Tracing]] — 运行 trace、hooks、结构化事件。
- [[harness/observability/replay/index|Replay]] — 基于轨迹复现 agent 行为。
- [[harness/observability/evals/index|Evals]] — harness 组件评测、ablation、benchmark。
- [[harness/observability/failure-taxonomy/index|Failure Taxonomy]] — 失败类型、根因分类。

### Research

- [[harness/research/index|Research]] — 外部材料的发现、分流与概念化入口；执行工作流在 automation 目录，不在 wiki 内。

## 目录约定

```text
concept/
  index.md
  implementations/
    uxarts-agent.md
    other-project.md
```

新增外部材料时，优先落到已有概念目录；只有找不到合适位置时，再新增概念。
