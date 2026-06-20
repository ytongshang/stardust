---
title: Memory Retrieval
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - memory
---

# Memory Retrieval

Memory Retrieval 是在正确时机找到相关记忆并注入上下文的过程。它可以是关键词、语义检索、LLM 选择或文件系统探索。

## 它解决什么问题

Long-Term Memory 只有在相关时才有价值。Memory Retrieval 负责在当前 task、workspace 和 user request 下找出真正有用的 memory，避免把一堆历史偏好无差别塞进 context。

它需要回答：

- 什么时候 retrieval：run start、user input 后、tool 前、compaction 后。
- 检索 query 如何构造：来自 user request、workspace state、task goal、agent profile。
- 相关性、recency、authority 和 privacy 如何排序。
- 找到的 memory 以什么形式进入 Context Injection。
- retrieval 失败或召回错误时如何观测和改进。

## 常见做法

常见实现包括：

- keyword / tag retrieval。
- semantic retrieval with embeddings。
- LLM rerank 或 LLM selection。
- scoped retrieval：按 user/project/workspace/team 限定。
- retrieval trace：记录 query、candidates、selected memories 和注入原因。

## 边界

- [[harness/memory/long-term-memory/index|Long-Term Memory]] 是存储；Memory Retrieval 是选择。
- [[harness/context/context-injection/index|Context Injection]] 决定 memory 何时、以什么 block 进入模型。
- [[harness/context/context-budgeting/index|Context Budgeting]] 决定 retrieval 结果能占多少 token。

## 目录

- [[harness/memory/memory-retrieval/implementations/uxarts-agent|uxarts-agent 实现]]
