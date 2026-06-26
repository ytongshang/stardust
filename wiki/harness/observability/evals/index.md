---
title: Evals
type: concept
created: 2026-06-20
updated: 2026-06-21
tags:
  - agent-harness
  - eval
---

# Evals

Evals 评估 harness 组件是否提升任务成功率、稳定性、成本、恢复能力或可观测性。

## 它解决什么问题

Harness 改动经常影响复杂行为：tool policy 变了、context 注入变了、compaction 变了，最终是否更好不能只靠感觉。Evals 用可重复任务、指标和 judge 来评估改动效果。

它需要衡量：

- task success：是否完成目标、产物是否正确。
- reliability：retry、resume、tool failure 后是否恢复。
- cost/latency：token、tool calls、wall time。
- safety：permission、risk control、sandbox policy 是否生效。
- observability：失败是否能被 trace 和 taxonomy 解释。

## 常见做法

常见 eval 形态包括：

- scenario tests：固定任务和 workspace。
- trace replay eval：历史 run 回放比较。
- change-manifest eval：每个 harness 改动先声明 expected fixes / regression risks，下一轮用 task-level delta 验证。
- model judge：用 LLM 评估 final answer 或 artifact。
- unit eval：只测 tool schema、prompt assembly、compaction summary。
- regression dashboard：按 harness concept 追踪指标变化。

## 边界

- [[harness/observability/replay/index|Replay]] 提供复现机制；Evals 定义任务、metric 和判定。
- [[harness/observability/failure-taxonomy/index|Failure Taxonomy]] 帮助把 eval failure 分类。
- [[harness/observability/tracing/index|Tracing]] 提供 eval debug 所需证据。
- [[harness/skills/skill-evaluation/index|Skill Evaluation]] 使用 eval 方法衡量 skill utility，但核心对象是 skill，归在 Skills 下。

## 目录

- [[harness/observability/evals/implementations/agentic-harness-engineering|Agentic Harness Engineering 实现]]
- [[harness/observability/evals/implementations/uxarts-agent|uxarts-agent 实现]]
