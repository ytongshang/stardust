---
title: Background Agents
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - multi-agent
---

# Background Agents

Background Agents 是独立异步运行的 agent。它们通常有自己的 session/workspace/lifecycle，可以在后台完成任务并回投结果。

## 它解决什么问题

有些任务不适合阻塞主对话，例如长时间搜索、构建验证、批量代码阅读、生成多个候选方案或等待外部系统。Background Agents 让这些工作可以独立运行，同时保持可追踪、可取消和可回收。

它需要回答：

- background agent 的启动 contract 和权限是什么。
- 它和主 agent 是否共享 workspace，还是使用 branch/isolated workspace。
- 用户如何看到 progress、cancel、resume 或接收结果。
- 完成后结果如何回投：message、artifact、diff、callback、notification。
- 主 agent 继续执行时如何感知 background result。

## 常见做法

常见实现包括：

- background run queue：独立排队和执行。
- progress events：把 status 写入 Event Log 或 notification channel。
- callback / completion hook：完成后通知 parent run 或产品层。
- isolated workspace：后台修改先在分支中保存，再 merge。
- lease / heartbeat：检测后台 run 是否失活。

## 边界

- [[harness/multi-agent/subagents/index|Subagents]] 可以同步或异步；Background Agents 特指异步独立 lifecycle。
- [[harness/lifecycle/queueing/index|Queueing]] 和 [[harness/lifecycle/heartbeat-retry/index|Heartbeat & Retry]] 支撑后台运行可靠性。
- [[harness/multi-agent/coordination/index|Coordination]] 处理多个 background agents 的分配和汇总。

## 目录

- [[harness/multi-agent/background-agents/implementations/uxarts-agent|uxarts-agent 实现]]
