---
title: Artifacts
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - workspace
---

# Artifacts

Artifacts 是 agent 产出的持久结果，例如代码文件、页面、设计产物、附件、发布组件或 application documents。

## 它解决什么问题

Agent 的结果不只是一条 final message。很多任务真正的输出是文件、diff、页面、图片、报告、数据集或可运行项目。Artifacts 让这些结果有独立身份、存储位置、版本和展示方式。

它需要表达：

- artifact type、owner、source run、created/updated time。
- 存储位置：workspace path、object storage、database、external URL。
- 展示方式：preview、download、diff、thumbnail、embed。
- 和 tool result、Event Log、final answer 的引用关系。
- 是否可被后续 run 继续读取、修改或发布。

## 常见做法

常见实现会把 artifact 和 message 分开：

- tool 生成 artifact，Tool Results 只返回摘要和 refs。
- Event Log 记录 artifact refs，而不是塞完整大内容。
- UI 根据 artifact metadata 渲染 preview。
- workspace snapshot 或 version history 记录 artifact 变化。
- sub-agent 产物通过 Workspace Merge 或 artifact merge 回到主上下文。

## 边界

- [[harness/tooling/tool-results/index|Tool Results]] 是一次 tool call 的返回 envelope；Artifacts 是可长期引用的结果。
- [[harness/workspace/workspace-model/index|Workspace Model]] 定义 artifact 存在哪里、如何访问。
- [[harness/workspace/snapshots/index|Snapshots]] 记录 artifact 在时间上的状态。

## 目录

- [[harness/workspace/artifacts/implementations/uxarts-agent|uxarts-agent 实现]]
