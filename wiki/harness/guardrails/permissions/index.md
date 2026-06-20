---
title: Permissions
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - guardrails
---

# Permissions

Permissions 定义 agent 在某个上下文下能访问哪些工具、文件、外部资源和危险操作。

## 它解决什么问题

Agent 的能力来自 tools、workspace、credentials 和 external services。Permissions 把能力边界显式化，确保模型不能因为 prompt 或 tool call 意图就越过用户、项目或系统授权。

它需要控制：

- tool access：哪些 tools 可见、可调用。
- resource access：文件、workspace、artifact、database、account。
- operation risk：read、write、delete、publish、payment、network。
- scope：user、project、workspace、organization、session。
- denied / approval required 时如何反馈给 agent 和 user。

## 常见做法

常见实现包括：

- policy engine：根据 context 判断 allow/deny/ask。
- capability token：给某次 run 临时授权。
- scoped credentials：只暴露当前任务需要的 secret。
- audit log：记录 sensitive action 和 permission decision。
- least privilege：默认最小权限，按需升级。

## 边界

- [[harness/tooling/tool-permissions/index|Tool Permissions]] 是 tool 层权限；Permissions 是整体 guardrails 视角。
- [[harness/guardrails/approvals-hitl/index|Approvals & HITL]] 是 permission escalation 的一种方式。
- [[harness/tooling/tool-availability/index|Tool Availability]] 可以根据 permission 隐藏 tools。

## 目录

- [[harness/guardrails/permissions/implementations/uxarts-agent|uxarts-agent 实现]]
