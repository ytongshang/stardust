---
title: Heartbeat & Retry
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - lifecycle
---

# Heartbeat & Retry

Heartbeat & Retry 用于长任务 liveness 检测和失败后恢复。它通常包含 lease、heartbeat timestamp、retry decision 和 delay queue。

## 它解决什么问题

Agent run 可能因为 worker crash、network timeout、model/tool hang、machine sleep 或外部服务错误而中断。Heartbeat & Retry 让系统能判断 run 是否仍然活着，以及失败后是否应该恢复、重试或标记失败。

它需要回答：

- 谁持有 run lease，lease 多久过期。
- heartbeat 多久更新一次，什么算 stale。
- stale run 是 retry、resume、cancel 还是等待人工处理。
- retry 是否幂等，是否会重复执行副作用 tool。
- 重试次数、backoff、dead-letter queue 如何控制。

## 常见做法

常见机制包括：

- lease token：worker 获取处理权。
- heartbeat timestamp：长任务定期续约。
- retry state：attempt count、last error、next retry time。
- idempotency key：防止 tool/action 重复落地。
- recovery worker：扫描 stale runs 并重新入队。

## 边界

- [[harness/lifecycle/durable-execution/index|Durable Execution]] 是总体恢复能力；Heartbeat & Retry 是其中的 liveness 和 retry 机制。
- [[harness/runtime/event-log/index|Event Log]] 用于恢复上下文，Retry 决定何时重新执行。
- [[harness/tooling/tool-error-recovery/index|Tool Error Recovery]] 处理 tool-level failure；Heartbeat & Retry 处理 run/worker-level failure。

## 目录

- [[harness/lifecycle/heartbeat-retry/implementations/uxarts-agent|uxarts-agent 实现]]
