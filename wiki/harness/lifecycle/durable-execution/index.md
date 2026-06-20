---
title: Durable Execution
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - lifecycle
  - durability
---

# Durable Execution

Durable Execution 是让长任务在模型调用、工具执行、机器重启、超时或中断后仍可恢复的能力。

## 它解决什么问题

Agent tasks 经常比一次 HTTP request 更长。Durable Execution 让 run 不依赖单个进程的内存：worker crash、机器 sleep、deploy、timeout 后，系统仍能从持久化状态恢复。

它需要具备：

- 持久 Event Log 和 queue state。
- 可恢复的 runtime state、pending state、workspace snapshot。
- idempotent tool/action 设计，避免重复副作用。
- heartbeat / lease / retry 机制。
- 明确 terminal state，避免 run 永远卡在 running。

## 常见做法

常见实现包括：

- persisted run record：status、attempts、owner worker、timestamps。
- append-only Event Log：恢复 messages 和 tool history。
- durable queue：worker 从数据库或 queue system 领取任务。
- checkpoints：workspace snapshot、compact summary、progress marker。
- recovery scanner：发现 stale runs 并重新入队。

## 边界

- [[harness/lifecycle/heartbeat-retry/index|Heartbeat & Retry]] 是 Durable Execution 的一个机制。
- [[harness/lifecycle/pending-resume/index|Pending & Resume]] 是有意挂起；Durable Execution 还覆盖非预期中断。
- [[harness/observability/replay/index|Replay]] 用同一批持久化数据复现行为，但 replay 目标是分析，不是继续运行。

## 目录

- [[harness/lifecycle/durable-execution/implementations/uxarts-agent|uxarts-agent 实现]]
