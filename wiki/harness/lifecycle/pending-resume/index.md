---
title: Pending & Resume
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - lifecycle
  - resume
---

# Pending & Resume

Pending & Resume 处理 agent 主动挂起、等待外部输入，再从保存的事件流继续运行。

## 它解决什么问题

有些 run 不能一次执行到底：需要 user approval、缺少信息、等待 webhook、等待 background agent 或等待长 tool 完成。Pending & Resume 让 run 可以显式进入等待状态，之后从保存的 state 继续，而不是重新开一个不连贯的任务。

它需要保留：

- pending reason 和等待对象。
- resume token / pending id / correlation id。
- 当前 Event Log、workspace snapshot、runtime state。
- 等待期间新输入如何合并或排队。
- resume 后从哪个点继续 Agent Loop。

## 常见做法

常见流程是：

- run 产生 pending event，停止当前 worker。
- lifecycle controller 保存 resume state。
- 外部输入、approval 或 callback 到达。
- resume event 入队，重新构建 context。
- Agent Loop 消费补充信息并继续执行。

## 边界

- [[harness/tooling/tool-interruption/index|Tool Interruption]] 是由 tool call 引发的 pending；Pending & Resume 是 run-level lifecycle。
- [[harness/guardrails/approvals-hitl/index|Approvals & HITL]] 是常见 pending source。
- [[harness/lifecycle/queueing/index|Queueing]] 决定 resume item 何时重新执行。

## 目录

- [[harness/lifecycle/pending-resume/implementations/uxarts-agent|uxarts-agent 实现]]
