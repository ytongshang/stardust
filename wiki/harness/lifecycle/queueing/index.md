---
title: Queueing
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - lifecycle
---

# Queueing

Queueing 处理 agent run 的并发控制、等待、会话内顺序和多轮输入合并。

## 它解决什么问题

同一个 user/session/workspace 可能连续发来多个 inputs，也可能同时有 background agents、tool callbacks 和 retry tasks。Queueing 让 harness 能控制执行顺序、并发数量和输入合并，避免 run 互相覆盖 workspace 或打乱 context。

它需要回答：

- 同一个 session 是否允许并发 run。
- 新 user input 到来时，是打断当前 run、排队、合并还是创建新 run。
- background run 和 foreground run 是否共享队列。
- queue item 的 priority、deadline、dedupe key 如何定义。
- 机器重启后 queue state 如何恢复。

## 常见做法

常见实现包括：

- per-session queue：保证同一 session 顺序执行。
- lease / lock：避免多个 worker 同时处理同一个 run。
- coalescing：把等待期间的新 input 合并到下一轮。
- priority queue：foreground、resume、retry、background 不同优先级。
- persisted queue：queue item 写入数据库，支持 durable execution。

## 边界

- [[harness/lifecycle/heartbeat-retry/index|Heartbeat & Retry]] 处理失活和重试；Queueing 处理进入执行前的排序。
- [[harness/runtime/agent-loop/index|Agent Loop]] 是 run 内循环；Queueing 管 run 之间的调度。
- [[harness/lifecycle/cancellation/index|Cancellation]] 可能清理 queued 或 running items。

## 目录

- [[harness/lifecycle/queueing/implementations/uxarts-agent|uxarts-agent 实现]]
