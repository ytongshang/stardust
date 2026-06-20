---
title: Cancellation
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - lifecycle
  - cancellation
---

# Cancellation

Cancellation 处理用户或系统停止 agent run，包括普通停止、软停止、队列清理和子 agent 停止。

## 它解决什么问题

Agent run 可能需要被用户停止、系统超时、policy 阻断或 parent run 取消。Cancellation 让停止成为可控 lifecycle，而不是简单 kill worker，避免 workspace 半写入、sub-agent 继续跑、queue 残留和状态不一致。

它需要回答：

- cancellation 是 soft stop、hard stop、timeout、policy stop 还是 user stop。
- running tool、background agent、pending callbacks 是否一起取消。
- 已产出的 artifacts 和 partial results 是否保留。
- workspace 写入是否需要回滚、提交或标记 partial。
- cancellation event 如何通知 UI、parent agent 和 queue。

## 常见做法

常见机制包括：

- cancellation token：Agent Loop 和 tool runner 定期检查。
- stop event：写入 Event Log，成为可审计终态。
- child propagation：向 subagents/background agents 传播取消。
- cleanup hooks：释放 lock、停止 sandbox、清理 queue item。
- partial result policy：明确取消后哪些输出可见。

## 边界

- [[harness/lifecycle/completion/index|Completion]] 处理包括 cancelled 在内的终态收尾。
- [[harness/tooling/tool-execution/index|Tool Execution]] 需要尊重 cancellation token。
- [[harness/multi-agent/background-agents/index|Background Agents]] 需要自己的 cancellation propagation。

## 目录

- [[harness/lifecycle/cancellation/implementations/uxarts-agent|uxarts-agent 实现]]
