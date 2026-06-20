---
title: Memory Persistence
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - memory
---

# Memory Persistence

Memory Persistence 是把记忆保存、更新、删除并在后续 session 中恢复的机制。

## 它解决什么问题

Memory 如果不能可靠 persistence，就只是一次 context injection。Memory Persistence 负责把可长期复用的信息保存下来，并在后续 run 中以可控方式恢复。

它需要处理：

- 写入条件：自动写、用户确认、显式命令、任务结束归档。
- 更新语义：append、overwrite、merge、expire、delete。
- scope：user、project、workspace、team、organization。
- provenance：这条 memory 来自哪个 run、哪段 evidence、谁确认过。
- privacy：敏感信息是否允许保存，如何删除和审计。

## 常见做法

常见实现包括：

- memory store：key-value、document store、vector store、file-backed memory。
- write policy：哪些事实可写，哪些必须 confirmation。
- conflict resolution：新旧 memory 冲突时保留哪个。
- retention policy：过期、归档、用户删除。
- audit trail：记录 memory 的创建、更新、使用和删除。

## 边界

- [[harness/memory/long-term-memory/index|Long-Term Memory]] 是概念集合；Memory Persistence 是读写生命周期。
- [[harness/memory/memory-retrieval/index|Memory Retrieval]] 使用 persisted memory，但不负责保存。
- [[harness/guardrails/risk-control/index|Risk Control]] 可能限制敏感 memory 的保存和使用。

## 目录

- [[harness/memory/memory-persistence/implementations/uxarts-agent|uxarts-agent 实现]]
