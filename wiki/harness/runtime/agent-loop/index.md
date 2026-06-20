---
title: Agent Loop
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - runtime
---

# Agent Loop

Agent Loop 是 agent runtime 的核心执行循环：准备输入、调用模型、解析输出、执行工具、处理 handoff/transfer/interrupt，并决定继续还是结束。

## 它解决什么问题

Agent Loop 把一次 agent run 从“单次 LLM call”提升为可持续推进的 runtime。它负责把模型输出、tool call、tool result、外部输入和终止条件串成一个可控循环。

它需要回答：

- 每轮开始前如何准备 messages、tools、runtime state 和 guardrails。
- 模型输出是 final answer、tool call、handoff、interrupt 还是异常。
- tool call 执行后如何把 result 放回下一轮上下文。
- 什么时候继续，什么时候 stop，什么时候 pending/resume。
- max turns、timeout、cancellation、failure recovery 如何影响循环。

## 常见做法

常见实现会把 loop 拆成几个明确步骤：

- prepare：读取 event log、workspace、memory、context blocks，生成本轮模型输入。
- model step：调用 model，记录 raw response、usage、tool calls、finish reason。
- dispatch：执行 tool、handoff 到其他 agent，或进入 pending state。
- observe：把 tool result、error、interrupt result 写回 event log。
- decide：根据 terminal condition、budget、queue、user input 决定下一轮。

好的 Agent Loop 通常不会把这些步骤都写死在一段 while loop 里，而是让每个步骤有可观测的输入输出，方便 replay、eval 和 failure analysis。

## 边界

- [[harness/runtime/event-log/index|Event Log]] 记录 loop 的事实历史；Agent Loop 决定下一步怎么走。
- [[harness/tooling/tool-execution/index|Tool Execution]] 负责跑 tool；Agent Loop 负责什么时候调用它以及如何消费结果。
- [[harness/lifecycle/pending-resume/index|Pending & Resume]] 是 loop 的挂起/恢复状态；不是一个独立对话流程。

## 目录

- [[harness/runtime/agent-loop/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部项目如果有不同的 loop/graph/runtime 边界，可以在本目录下新增实现笔记。重点看它如何处理多 turn、工具执行、挂起恢复、handoff、max turn、streaming 和 failure recovery。
