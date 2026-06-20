---
title: Working Memory
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - memory
---

# Working Memory

Working Memory 是当前任务/会话内的短期状态，例如当前目标、任务列表、中间推理、最近工具结果和未完成事项。

## 它解决什么问题

Agent run 需要维护一组“现在正在发生什么”的状态，但这些状态不一定都适合写进 Long-Term Memory，也不一定都应该永远保留在 prompt 里。Working Memory 让当前任务的目标、计划、progress、constraints 和临时事实有明确位置。

它需要回答：

- 当前目标和 success criteria 是什么。
- 已完成、正在做、待确认的事项是什么。
- 哪些 tool results 是后续步骤需要引用的临时事实。
- 哪些状态应该在 compaction 后保留。
- 哪些状态只属于当前 run，不应该跨 session 保存。

## 常见做法

常见实现包括：

- task state：todo list、current step、blocked reason。
- scratchpad：中间 notes、hypotheses、temporary ids。
- recent observations：近期 tool results 的摘要和 refs。
- progress markers：checkpoint、last successful action、next action。
- compact summary：把 working state 写进 summary，避免长任务断片。

## 边界

- [[harness/memory/long-term-memory/index|Long-Term Memory]] 跨 session；Working Memory 通常只服务当前 run。
- [[harness/context/compaction/index|Compaction]] 会把 Working Memory 摘要化，但不等于 memory system。
- [[harness/runtime/event-log/index|Event Log]] 记录事实历史，Working Memory 是从事实中提炼出的当前状态视图。

## 目录

- [[harness/memory/working-memory/implementations/uxarts-agent|uxarts-agent 实现]]
