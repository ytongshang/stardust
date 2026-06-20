---
title: Tool Interruption
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
  - interrupt
---

# Tool Interruption

Tool Interruption 是工具层的挂起机制：模型发起工具调用，但工具需要等待用户、另一个 agent 或外部系统补充结果。

## 它解决什么问题

有些 tool call 不能立即返回：需要用户选择、等待后台任务、等待外部 webhook、等待另一个 agent 完成。Tool Interruption 让 tool execution 可以安全挂起，而不是用长轮询、阻塞线程或让模型反复追问。

它需要保留：

- 原始 tool call 和参数。
- 等待对象：user、external system、background agent、approval。
- resume token / pending id。
- 挂起期间如何展示状态、处理 cancellation 和 timeout。
- 恢复后 result 如何接回原来的 Agent Loop。

## 常见做法

常见实现会把 interrupted tool result 记录成一种特殊 Tool Result：

- tool runner 返回 pending/interrupted envelope。
- lifecycle controller 持久化 resume state。
- 外部输入到达后生成 resume event。
- Agent Loop 从 Event Log 中恢复上下文，并把补充结果作为 tool observation。

## 边界

- [[harness/lifecycle/pending-resume/index|Pending & Resume]] 是 run 级别的挂起/恢复；Tool Interruption 是由 tool call 触发的 pending。
- [[harness/guardrails/approvals-hitl/index|Approvals & HITL]] 是常见 interruption 来源，但 interruption 也可以等待系统任务。
- [[harness/tooling/tool-results/index|Tool Results]] 需要能表达 interrupted/pending 状态。

## 目录

- [[harness/tooling/tool-interruption/implementations/uxarts-agent|uxarts-agent 实现]]
