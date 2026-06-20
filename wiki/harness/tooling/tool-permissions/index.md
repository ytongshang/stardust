---
title: Tool Permissions
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
  - permissions
---

# Tool Permissions

Tool Permissions 定义某个 agent profile、task state 或 execution context 下，工具能否被调用、能访问哪些资源、能写哪些路径、是否需要 approval。

## 它解决什么问题

Tool Permissions 把“模型想做什么”和“系统允许做什么”分开。即使模型看到了 tool，runtime 仍需要在执行前确认资源边界和风险边界。

它需要控制：

- 文件系统读写范围、workspace scope、artifact scope。
- 网络、shell、browser、credentials、external API 访问。
- destructive action、payment、publish、delete 等高风险操作。
- user/session/project/team 维度的授权差异。
- permission denied 后给模型和用户的反馈。

## 常见做法

常见实现会把 permissions 分成多层：

- static policy：由 agent type、tool type、environment 决定的默认权限。
- runtime grant：用户或系统在某次 run 中授予的临时权限。
- resource scope：路径、host、account、project、dataset 等资源边界。
- approval gate：高风险操作前暂停并等待 HITL。
- audit record：记录谁在什么时候允许了什么操作。

## 边界

- [[harness/guardrails/permissions/index|Permissions]] 是更通用的安全概念；Tool Permissions 是其中作用在 tool 层的部分。
- [[harness/tooling/tool-availability/index|Tool Availability]] 可以隐藏无权限 tool；Tool Permissions 仍需要在 execution 前强制检查。
- [[harness/guardrails/sandbox-policy/index|Sandbox Policy]] 定义隔离环境能力，通常和 tool permission 一起生效。

## 目录

- [[harness/tooling/tool-permissions/implementations/uxarts-agent|uxarts-agent 实现]]
