---
title: Snapshots
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - workspace
---

# Snapshots

Snapshots 记录工作区在某个时间点的状态，用于历史视图、回滚、diff、复制工作区或恢复任务。

## 它解决什么问题

长任务和多 agent 场景里，workspace 会不断变化。Snapshots 让 harness 能回答“某一刻 workspace 是什么样”，从而支持 diff、rollback、branch、replay 和用户审计。

它需要记录：

- snapshot scope：整个 workspace、某些 artifacts、文件树还是 metadata。
- snapshot trigger：run start、tool write、completion、handoff、manual checkpoint。
- snapshot storage：copy、content-addressed objects、git commit、database version。
- snapshot relation：parent、branch、merge base、run id。
- restore semantics：恢复时是否覆盖当前状态，如何处理冲突。

## 常见做法

常见实现包括：

- git commit/worktree snapshot。
- file tree manifest + content hash。
- artifact version record。
- database transaction checkpoint。
- lightweight checkpoint，只记录 changed files 和 metadata。

## 边界

- [[harness/workspace/workspace-merge/index|Workspace Merge]] 需要 snapshot 或 common base 来检测冲突。
- [[harness/observability/replay/index|Replay]] 需要 snapshot 还原当时 workspace。
- [[harness/lifecycle/durable-execution/index|Durable Execution]] 依赖 snapshot 或 event log 来恢复长任务。

## 目录

- [[harness/workspace/snapshots/implementations/uxarts-agent|uxarts-agent 实现]]
