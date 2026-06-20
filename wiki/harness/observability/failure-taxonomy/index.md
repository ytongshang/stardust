---
title: Failure Taxonomy
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - observability
  - failure
---

# Failure Taxonomy

Failure Taxonomy 是对 agent 失败原因的结构化分类，例如模型行为、工具参数、权限、上下文丢失、workspace 冲突、生命周期超时等。

## 它解决什么问题

如果每次失败都只写“agent failed”，就无法判断应该改 prompt、tool、context、lifecycle 还是 product policy。Failure Taxonomy 给失败原因一组稳定分类，让 debug、eval、监控和 roadmap 都能对齐。

它需要区分：

- model failure：误解任务、错误推理、无效 tool choice。
- context failure：缺少关键信息、compaction 丢事实、memory 召回错。
- tool failure：schema error、runtime exception、unavailable tool。
- workspace failure：conflict、missing file、bad merge、sandbox mismatch。
- lifecycle failure：timeout、stale run、resume lost、queue race。
- guardrail failure：permission denied、risk block、approval missing。

## 常见做法

常见实现包括：

- finite category set + subcategory。
- failure tags 写入 trace/eval result。
- root cause 和 immediate cause 分开。
- retryable / non-retryable 标记。
- taxonomy-driven dashboard：按类别看失败率和趋势。

## 边界

- [[harness/observability/tracing/index|Tracing]] 记录证据；Failure Taxonomy 给证据命名。
- [[harness/tooling/tool-error-recovery/index|Tool Error Recovery]] 可以使用 taxonomy 决定重试策略。
- [[harness/observability/evals/index|Evals]] 用 taxonomy 聚合失败模式。

## 目录

- [[harness/observability/failure-taxonomy/implementations/uxarts-agent|uxarts-agent 实现]]
