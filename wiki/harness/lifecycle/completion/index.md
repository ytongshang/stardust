---
title: Completion
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - lifecycle
---

# Completion

Completion 处理 agent run 的终态、完成回调、事件广播和后续队列/压缩/回投动作。

## 它解决什么问题

Agent run 结束时不只是输出一段 final answer。Completion 负责把 run 的终态、产物、回调、通知、清理和后续任务统一收尾，避免运行结束后留下半成品状态。

它需要回答：

- run 是 success、failed、cancelled、partial、pending timeout 还是 transferred。
- final answer、artifacts、workspace changes 如何提交。
- parent run、background callback、UI、notification 如何收到结果。
- 是否触发 compaction、memory write、eval、trace export。
- queued follow-up inputs 是否继续执行。

## 常见做法

常见 completion handler 会做：

- 写入 terminal event。
- flush tool results、artifacts、trace spans。
- release lease / lock。
- broadcast status to UI 或 parent agent。
- schedule post-run jobs：summary、memory persistence、eval、cleanup。

## 边界

- [[harness/runtime/agent-loop/index|Agent Loop]] 决定何时到达终态；Completion 负责终态后的收尾。
- [[harness/workspace/artifacts/index|Artifacts]] 记录产物；Completion 决定何时提交和回投。
- [[harness/observability/tracing/index|Tracing]] 和 [[harness/observability/evals/index|Evals]] 常在 completion 后落地。

## 目录

- [[harness/lifecycle/completion/implementations/uxarts-agent|uxarts-agent 实现]]
