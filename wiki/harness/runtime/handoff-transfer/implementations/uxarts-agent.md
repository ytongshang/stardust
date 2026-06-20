---
title: uxarts-agent — Handoff & Transfer
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/agent_runner.py
  - uxarts-agent/uxarts/core/agent/types/items.py
tags:
  - uxarts-agent
  - handoff
---

# uxarts-agent — Handoff & Transfer

uxarts-agent 在 `AgentRunner` 中区分 handoff 和 transfer。

- `NextStepHandoff`: 切换到新的 `UxAgent`，继续同一轮 agent loop。
- `NextStepTransfer`: final output 后的流程移交。
- `UxHandoffCallItem` / `UxHandoffOutputItem` / `UxHandoffInputItem`: handoff 相关事件。
- `UxFlowTransferItem`: flow transfer 事件。

这些事件都会进入 `RunItem` 历史，`GetuiRunner` 后续通过 `agent_id` / `previous_id` / item 顺序重建当前 agent。

当前没有独立的 handoff policy 层；handoff 和 transfer 主要由 agent 定义、model response parsing 和 runner next step 决定。
