---
title: Approvals & HITL
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - guardrails
  - hitl
---

# Approvals & HITL

Approvals & HITL 是让 agent 在关键节点等待用户或外部系统确认的机制。

## 它解决什么问题

有些动作不能只靠模型自行决定，例如删除文件、发布、付款、发送外部消息、访问敏感数据或选择不可逆方案。Approvals & HITL 让 agent 在高风险节点暂停，把决策权交回 user 或 policy owner。

它需要回答：

- 哪些动作需要 approval，触发条件是什么。
- approval request 展示什么上下文、风险和可选项。
- 用户批准、拒绝、修改后如何回到 Agent Loop。
- approval 是否有有效期、scope 和 audit record。
- 等待 approval 时 run 是 pending、cancelled 还是继续做其他事。

## 常见做法

常见实现包括：

- approval gate：tool execution 前拦截。
- structured request：action、risk、diff、resource、reason。
- resumable result：批准后继续执行原 tool call。
- delegated approval：组织 policy 或外部系统审批。
- audit trail：记录 request、decision、actor、timestamp。

## 边界

- [[harness/lifecycle/pending-resume/index|Pending & Resume]] 提供等待和恢复机制；Approvals & HITL 是其中一种等待原因。
- [[harness/guardrails/permissions/index|Permissions]] 决定是否需要 approval。
- [[harness/tooling/tool-interruption/index|Tool Interruption]] 可用来表达需要 approval 的 tool call。

## 目录

- [[harness/guardrails/approvals-hitl/implementations/uxarts-agent|uxarts-agent 实现]]
