---
title: Sandbox Policy
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - guardrails
  - sandbox
---

# Sandbox Policy

Sandbox Policy 定义 agent 生成代码或命令在哪些隔离环境执行、能访问哪些凭证和网络、失败如何回传。

## 它解决什么问题

Sandbox 本身只是执行环境，Sandbox Policy 决定这个环境能做什么。它防止 agent 运行的代码越界访问主机、网络、secret、用户文件或高成本资源。

它需要控制：

- filesystem mount、read/write paths、artifact sync。
- network allowlist / denylist。
- secret injection 和 credential scope。
- CPU、memory、disk、time limit。
- command allowlist、package install policy、preview exposure。

## 常见做法

常见策略包括：

- per-run sandbox：每个 run 独立环境。
- restricted mounts：只挂载 workspace 子集。
- no-secret-by-default：需要 tool 或 approval 才注入 secret。
- network policy：默认禁网或只允许特定 domains。
- cleanup policy：run 完成后销毁或保留 snapshot。

## 边界

- [[harness/workspace/sandbox/index|Sandbox]] 是执行环境；Sandbox Policy 是限制规则。
- [[harness/guardrails/permissions/index|Permissions]] 决定谁能请求某类 sandbox capability。
- [[harness/tooling/tool-execution/index|Tool Execution]] 在执行 shell/code tools 时应用 policy。

## 目录

- [[harness/guardrails/sandbox-policy/implementations/uxarts-agent|uxarts-agent 实现]]
