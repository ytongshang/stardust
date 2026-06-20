---
title: Tracing
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - observability
---

# Tracing

Tracing 记录 agent run 中发生的事件、工具调用、模型调用、token、耗时和上下文信息。

## 它解决什么问题

Agent 失败时，只有 final answer 很难定位问题。Tracing 让 harness 能看到 run 内每一步发生了什么：model call、tool call、context assembly、permission decision、retry、handoff、cost 和 latency。

它需要记录：

- span hierarchy：run、model step、tool execution、retrieval、compaction、sub-agent。
- inputs/outputs 的摘要和 refs，不一定保存全部敏感内容。
- timing、token usage、cost、model、tool、worker。
- error、retry、cancellation、pending、resume 等 lifecycle events。
- trace id 与 Event Log、artifacts、eval sample 的关联。

## 常见做法

常见实现包括：

- OpenTelemetry-like spans。
- structured run trace。
- model/tool call logging。
- context snapshot 或 prompt block provenance。
- timeline UI：按时间展示 agent 行为。

## 边界

- [[harness/runtime/event-log/index|Event Log]] 是可恢复事实日志；Tracing 更偏观测和诊断。
- [[harness/observability/replay/index|Replay]] 用保存的数据复现；Tracing 帮助理解当时发生了什么。
- [[harness/observability/evals/index|Evals]] 可以从 traces 中抽样失败案例。

## 目录

- [[harness/observability/tracing/implementations/uxarts-agent|uxarts-agent 实现]]
