---
title: Coordination
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - multi-agent
---

# Coordination

Coordination 处理多 agent 的任务分配、fanout、依赖、归属、冲突和结果汇总。

## 它解决什么问题

当多个 agents 同时工作时，系统需要决定谁做什么、什么时候开始、如何汇总、冲突怎么处理。Coordination 防止多 agent 只是“并行跑几个对话”，而是形成可控的执行计划。

它需要回答：

- task 如何拆分，哪些可以 parallel，哪些有 dependency。
- 哪个 agent 拥有什么责任和输出 contract。
- fanout 数量、预算、超时、取消如何控制。
- partial results 如何聚合，冲突如何仲裁。
- coordination state 如何进入 trace 和 replay。

## 常见做法

常见策略包括：

- planner/coordinator：一个 agent 或 runtime 负责任务分配。
- task graph：显式建模 dependencies 和 status。
- fanout/fanin：并行探索后汇总结果。
- voting/judging：多个结果由 judge 或 rule 选择。
- shared workspace protocol：限制谁能写什么，何时 merge。

## 边界

- [[harness/multi-agent/teams/index|Teams]] 定义组织结构；Coordination 定义运行策略。
- [[harness/lifecycle/queueing/index|Queueing]] 负责执行排队；Coordination 负责任务关系和依赖。
- [[harness/observability/tracing/index|Tracing]] 应记录 coordination decisions，方便分析多 agent 失败。

## 目录

- [[harness/multi-agent/coordination/implementations/uxarts-agent|uxarts-agent 实现]]
