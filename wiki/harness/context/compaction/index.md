---
title: Compaction
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - compaction
  - context
---

# Compaction

Compaction 是 context window 不够时，对历史进行摘要、替换和恢复的机制。它包括触发、summary、trim item、恢复拼接和 post-compact context 注入。

## 它解决什么问题

长任务里 event history 会持续增长，但模型每轮只能看到有限 context。Compaction 让 harness 在不丢掉任务连续性的前提下，把旧历史压缩成可继续执行的 working summary。

它要保留的不只是聊天摘要，还包括：

- 当前目标、已完成事项、未完成事项和用户偏好。
- 已创建/修改的 artifacts、workspace paths 和关键决策。
- tool 调用得到的事实、错误和外部约束。
- pending 状态、handoff 关系和后续执行提醒。
- 压缩前后哪些 events 被 summary 替代。

## 常见做法

常见流程是：

- trigger：根据 token budget、turn count、tool output size 或 explicit command 触发。
- summarize：用模型或 deterministic rules 生成 compact summary。
- replace：把旧 events 标记为 compacted，保留 summary event 和必要 recent tail。
- reinject：把 static rules、workspace state、memory、current task 重新拼回 context。
- validate：用 trace/eval 检查 summary 是否保留关键事实和可执行状态。

## 目录

- [[harness/context/compaction/implementations/uxarts-agent|uxarts-agent 实现]]

## 边界

- [[harness/context/context-budgeting/index|Context Budgeting]] 决定是否需要压缩；Compaction 执行历史替换。
- [[harness/runtime/event-log/index|Event Log]] 保存压缩前后的事实关系；Compaction 不应该让历史不可审计。
- [[harness/observability/evals/index|Evals]] 可以评估 compact summary 是否影响任务成功率。

## 后续收集

外部项目如果有分层压缩、cache-safe compact、summary eval、trace replay 或 deterministic compression，可以在本目录下新增笔记。
