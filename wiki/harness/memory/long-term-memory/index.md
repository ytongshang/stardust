---
title: Long-Term Memory
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - memory
---

# Long-Term Memory

Long-Term Memory 是跨 session 保留的用户偏好、项目知识、经验和领域背景。

## 它解决什么问题

有些信息对后续任务长期有用，例如用户偏好、项目结构、常用约定、已验证的经验、团队规则和历史决策。Long-Term Memory 让这些知识不必每次都由用户重复提供。

它需要控制：

- 什么信息值得长期保存，什么只是当前 run 的临时状态。
- 记忆属于 user、project、workspace、team 还是 agent。
- 记忆如何被更新、覆盖、过期和删除。
- 记忆注入时如何避免污染当前任务或泄漏不相关信息。
- 用户如何查看、纠正和撤销 memory。

## 常见做法

常见实现包括：

- profile memory：用户偏好、语言、工作方式。
- project memory：代码结构、约定、常用命令、架构决策。
- episodic memory：过去任务的摘要、成功/失败经验。
- vector / keyword index：用于按需 retrieval。
- memory policy：决定什么自动保存，什么需要用户确认。

## 边界

- [[harness/memory/working-memory/index|Working Memory]] 是当前 run 的短期状态；Long-Term Memory 跨 run。
- [[harness/memory/memory-retrieval/index|Memory Retrieval]] 负责找出相关 memory；Long-Term Memory 是被检索的存储层和知识集合。
- [[harness/guardrails/permissions/index|Permissions]] 决定哪些 memory 可以被当前 agent 读取或写入。

## 目录

- [[harness/memory/long-term-memory/implementations/uxarts-agent|uxarts-agent 实现]]
