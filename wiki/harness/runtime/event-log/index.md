---
title: Event Log
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - runtime
  - event-log
---

# Event Log

Event Log 是 agent run 的可恢复事实日志。它记录用户输入、模型输出、工具调用、工具结果、handoff、压缩、结束事件等，让 harness 可以重建上下文、审计历史、做 replay/eval。

## 它解决什么问题

没有 Event Log，agent run 很容易变成只存在于内存里的临时过程：失败后无法恢复，线上问题无法定位，eval 无法复现，context 也很难可靠重建。

Event Log 的核心价值是把运行过程变成一条 append-only 或接近 append-only 的事实序列：

- 重建 messages 和 runtime state。
- 支持 pending/resume、retry、compaction 和 handoff。
- 让 trace、replay、eval 有统一数据来源。
- 保留 tool call 与 tool result 的对应关系。
- 区分模型说过什么、tool 做过什么、系统补写过什么。

## 常见做法

常见 schema 会显式区分：

- user input / transformed input。
- model response / assistant message / tool call。
- tool result / error / interrupted result。
- lifecycle event，例如 queued、started、pending、resumed、completed、cancelled。
- context event，例如 compaction、context injection、memory retrieval。

关键不只是“把 message 存起来”，而是保留 event type、ordering、source、idempotency key、parent call id、timestamp 和可用于 replay 的 metadata。

## 边界

- Event Log 记录事实，不负责决定下一步执行；下一步由 [[harness/runtime/agent-loop/index|Agent Loop]] 或 lifecycle controller 决定。
- [[harness/observability/tracing/index|Tracing]] 可以从 Event Log 派生，但通常会加入耗时、span、cost 等观测字段。
- [[harness/observability/replay/index|Replay]] 依赖 Event Log，但还需要 model/tool/config 的可复现策略。

## 目录

- [[harness/runtime/event-log/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部项目如果有 trajectory schema、message ledger、trace event、replay log 或 invariant 校验，可以放到本目录。
