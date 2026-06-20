---
title: Workspace Model
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - workspace
---

# Workspace Model

Workspace Model 定义 agent 可操作的工作区是什么：本地文件系统、虚拟文件系统、产品 artifact backend、git worktree、sandbox 或这些的组合。

## 它解决什么问题

Agent 需要一个可读、可写、可验证的工作对象。Workspace Model 明确 agent 操作的是文件、artifact、database、browser state、remote sandbox 还是这些的组合，并定义它们之间如何映射。

它需要回答：

- workspace 的根是什么，路径和 artifact id 如何对应。
- read/write/delete/list/search/diff 等基本操作由谁提供。
- workspace 是否支持 transaction、snapshot、branch、merge。
- agent、tool、sub-agent 看到的是同一个 workspace 还是隔离视图。
- workspace state 如何进入 context、trace、replay 和 UI。

## 常见做法

常见模型包括：

- local filesystem：适合代码编辑、测试和构建。
- virtual filesystem：适合浏览器内或远端环境。
- artifact backend：适合产品化产物、设计文件、页面、文档。
- git worktree：适合 branch、diff、merge 和 review。
- sandbox workspace：适合不信任代码执行和临时实验。

## 边界

- [[harness/workspace/artifacts/index|Artifacts]] 是 workspace 中值得持久化/展示的结果。
- [[harness/workspace/snapshots/index|Snapshots]] 记录 workspace 的某个时间点。
- [[harness/guardrails/sandbox-policy/index|Sandbox Policy]] 限制 workspace 操作的执行环境和权限。

## 目录

- [[harness/workspace/workspace-model/implementations/uxarts-agent|uxarts-agent 实现]]
