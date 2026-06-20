---
title: uxarts-agent — Coordination
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/task
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/subagent
tags:
  - uxarts-agent
  - coordination
---

# uxarts-agent — Coordination

uxarts-agent 的 coordination 主要通过 task manager 和 background sub-agent lifecycle 实现：

- `TaskUpdate(owner=agent_id)` 把任务和子 agent 绑定。
- fanout limit 控制后台 agent 数量。
- launch idempotency 依赖 tool_call_item_id / sub index。
- completion listener 决定是继续子队列还是回投父 session。
- artifact merge 由主 agent 显式触发。

当前没有通用 DAG scheduler；coordination 主要是产品化任务归属和生命周期规则。
