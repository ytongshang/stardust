---
title: Replay
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - observability
  - replay
---

# Replay

Replay 是基于保存的事件流、上下文和配置复现 agent 行为，用于调试、回归和 eval。

## 它解决什么问题

Agent 行为受 model、prompt、tools、workspace、memory、runtime state 共同影响。Replay 让一次 run 可以被重放，用来定位 bug、比较改动、做 regression 和构建 eval cases。

它需要回答：

- replay 使用真实 model 还是 mock/stub model。
- tool calls 是重新执行、读取旧结果，还是 sandboxed re-execution。
- workspace、memory、skill version、tool definitions 如何恢复到当时状态。
- replay 是否 deterministic，哪些部分允许漂移。
- replay 结果如何和原 run diff。

## 常见做法

常见模式包括：

- event replay：按 Event Log 重建 messages 和 tool observations。
- golden trace：固定模型输出和 tool results 做 deterministic replay。
- fork replay：从某个 event 开始换新 prompt/model/tool policy。
- eval replay：批量跑历史失败案例看新 harness 是否改善。
- snapshot restore：恢复 workspace 后重新执行关键路径。

## 边界

- [[harness/runtime/event-log/index|Event Log]] 和 [[harness/workspace/snapshots/index|Snapshots]] 是 replay 输入。
- [[harness/observability/evals/index|Evals]] 使用 replay 做回归评估，但 eval 还需要 metric 和 judge。
- [[harness/lifecycle/durable-execution/index|Durable Execution]] 是继续真实任务；Replay 是分析/测试路径。

## 目录

- [[harness/observability/replay/implementations/uxarts-agent|uxarts-agent 实现]]
