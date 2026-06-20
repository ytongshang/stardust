---
title: Subagents
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - multi-agent
---

# Subagents

Subagent 是主 agent 把部分任务委托给另一个 agent 的机制。它可以是同步 inline 工具，也可以是独立 background session。

## 它解决什么问题

Subagent 让主 agent 可以把可分解的工作交给更窄职责、更独立上下文或更长生命周期的执行者。它常用于搜索、代码阅读、并行探索、专门工具操作和长任务拆分。

它需要回答：

- subagent 的 task contract 是什么：输入、约束、输出格式、完成标准。
- subagent 是否共享 parent 的 context、workspace、memory 和 tools。
- parent 是否阻塞等待，还是让 subagent background running。
- subagent 产生的 artifacts、trace 和 final result 如何回到 parent。
- subagent 失败、超时、被取消时 parent 如何恢复。

## 常见做法

常见实现包括：

- inline subagent：作为 tool call 同步运行，返回 summary/result。
- background subagent：独立 session / run，完成后 callback 或 message 回投。
- scoped context：只给 subagent 必要 context，避免污染或泄漏。
- child workspace：让 subagent 在隔离 workspace 中产出 diff/artifacts。
- parent-child trace：保留任务树和 result merge 关系。

## 边界

- [[harness/runtime/handoff-transfer/index|Handoff & Transfer]] 是执行主体切换；Subagents 是其中一种多 agent 形态。
- [[harness/multi-agent/background-agents/index|Background Agents]] 强调异步 lifecycle，可能由 subagent 机制创建。
- [[harness/workspace/workspace-merge/index|Workspace Merge]] 处理 subagent 产物回到主 workspace。

## 目录

- [[harness/multi-agent/subagents/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部项目如果有新的 sub-agent lifecycle、communication protocol、completion callback、artifact merge、fanout control 或 multi-agent trace，可以在本目录下新增笔记。
