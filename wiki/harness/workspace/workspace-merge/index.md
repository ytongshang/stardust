---
title: Workspace Merge
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - workspace
  - multi-agent
---

# Workspace Merge

Workspace Merge 处理子工作区或并行分支的产物如何回到主工作区，包括冲突检测、三路合并、人工/agent 解决冲突。

## 它解决什么问题

Sub-agent、background agent、branch workspace 或 sandbox experiment 可能并行修改同一批文件/ artifacts。Workspace Merge 决定这些修改如何安全回到主 workspace，避免覆盖用户或其他 agent 的工作。

它需要回答：

- merge 的 source、target、base 是什么。
- 冲突如何检测：文本 diff、artifact metadata、semantic conflict、path conflict。
- 谁解决冲突：自动合并、agent、user approval、domain-specific merge tool。
- merge 后如何更新 Event Log、artifact refs、snapshots 和 UI。
- merge 失败时 source workspace 是否保留、是否可重试。

## 常见做法

常见策略包括：

- three-way merge：适合代码和文本。
- artifact-level merge：按 artifact id/version 合并。
- patch proposal：子 agent 提交 diff，由主 agent 或 user 接受。
- isolated branch：background task 在独立 workspace 中完成后再回投。
- conflict envelope：把冲突作为结构化 result 交给 agent/user 处理。

## 边界

- [[harness/multi-agent/subagents/index|Subagents]] 或 [[harness/multi-agent/background-agents/index|Background Agents]] 常常产生 merge 来源，但 merge 是 workspace 层概念。
- [[harness/workspace/snapshots/index|Snapshots]] 提供 merge base。
- [[harness/guardrails/approvals-hitl/index|Approvals & HITL]] 可用于高风险 merge 前确认。

## 目录

- [[harness/workspace/workspace-merge/implementations/uxarts-agent|uxarts-agent 实现]]
