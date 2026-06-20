---
title: Teams
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - multi-agent
---

# Teams

Teams 是多个 agent 以角色、职责或组织结构协作完成任务的抽象。它比单个 subagent 更强调角色分工和团队协议。

## 它解决什么问题

当任务需要多个持续角色，而不只是临时委托一个 subagent 时，Teams 提供更高层的组织抽象。它关注 role、membership、protocol、coordination policy 和 shared workspace。

它需要回答：

- team 中有哪些 roles，例如 planner、coder、reviewer、researcher、critic。
- 谁负责分配任务、仲裁冲突、合并结果。
- team members 是否共享 memory、workspace、tools 和 context。
- 团队协议是 sequential、parallel、debate、review loop 还是 manager-worker。
- team 的输出如何变成单一 final result。

## 常见做法

常见模式包括：

- manager-worker：一个 coordinator 分配任务给多个 workers。
- reviewer loop：builder 和 reviewer 交替迭代。
- debate / critique：多个 agents 提出观点，由 judge 汇总。
- specialist team：不同领域 agents 处理不同子问题。
- shared board：通过 task board、artifact store 或 message bus 协作。

## 边界

- [[harness/multi-agent/subagents/index|Subagents]] 是局部委托；Teams 是稳定组织结构。
- [[harness/multi-agent/coordination/index|Coordination]] 是 teams 的运行机制之一。
- [[harness/multi-agent/agent-communication/index|Agent Communication]] 是 teams 的通信层。

## 目录

- [[harness/multi-agent/teams/implementations/uxarts-agent|uxarts-agent 实现]]
