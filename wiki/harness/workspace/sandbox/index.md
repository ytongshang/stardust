---
title: Sandbox
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - workspace
  - sandbox
---

# Sandbox

Sandbox 是 agent 执行代码、测试、构建、预览或外部命令的隔离环境。它属于 workspace 和 guardrails 的交叉概念。

## 它解决什么问题

Agent 经常需要运行不完全可信的代码或命令。Sandbox 提供隔离的 execution environment，让 agent 可以验证结果，同时限制对主机、凭证、网络和用户文件的影响。

它需要控制：

- filesystem scope、network scope、process scope、secret scope。
- install/build/test/preview 命令能做什么。
- sandbox state 是否持久，是否和 workspace 同步。
- stdout/stderr、exit code、preview URL、generated files 如何回传。
- sandbox crash、timeout、resource limit 如何恢复。

## 常见做法

常见 sandbox 形态包括：

- local restricted shell。
- container / VM / cloud sandbox。
- browser sandbox 或 hosted preview。
- language-specific runtime，例如 Python/Node code interpreter。
- per-run or per-agent isolated workspace。

## 边界

- [[harness/guardrails/sandbox-policy/index|Sandbox Policy]] 定义 sandbox 的权限和限制；Sandbox 是执行环境本身。
- [[harness/tooling/tool-execution/index|Tool Execution]] 可以在 sandbox 中运行 tool。
- [[harness/workspace/workspace-model/index|Workspace Model]] 定义 sandbox 和主 workspace 如何同步文件。

## 目录

- [[harness/workspace/sandbox/implementations/uxarts-agent|uxarts-agent 实现]]
