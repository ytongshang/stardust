---
title: Run Ledger
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/types/items.py
  - uxarts-agent/uxarts/web/modules/getui/task/getui_runner.py
tags:
  - agent-harness
  - event-log
---

# Run Ledger

## 这个概念是什么

Run Ledger 是 agent run 的事实日志。它比普通 chat messages 更细：不仅记录 user/model message，也记录 agent start、tool call、tool output、handoff、trim、custom event。

通用价值：

- 可恢复。
- 可重放。
- 可审计。
- 可做 trace/eval。
- 可支持 pending/resume。
- 可从历史中重建当前 agent 状态。

## uxarts-agent 现在做什么

`RunItem` 是事实日志单位，center 持久化 item，Python 每轮从 item 流重建上下文。

主要 item：

- `UxAgentStartItem`
- `UxUserInputItem`
- `UxModelGenerationItem`
- `UxToolCallItem`
- `UxToolCallOutputItem`
- `UxHandoffCallItem`
- `UxHandoffOutputItem`
- `UxHandoffInputItem`
- `UxAgentEndItem`
- `UxFlowTransferItem`
- `UxMessageTrimItem`
- `UxCustomizeItem`

## uxarts-agent 怎么做

`MessageLoader` 把 center 的 `GetuiItemData` 解析成 Python `RunItem`。

重要逻辑：

- `process_status == 0` 时，如果有 `raw_input`，才调用 `user_input_creator` 生成 LLM message。
- 已处理的历史 user input 直接复用 `raw_message`，避免每轮重复拼接 context。
- agent start item 会恢复 `UxAgent`、`agent_id`、`previous_id`。

`GetuiRunner` 会：

- 按 agent_id 分组历史。
- 找到当前要继续运行的 agent。
- 扫描历史 tool call/output。
- 修补缺失 tool call/output 为 fake item。
- 上报 fake item，避免静默腐化。

## 不能丢的细节

- `agent_id + item_id` 是多 agent 历史结构的核心。
- `previous_id` 表达 agent 分支/继承关系。
- tool call 必须最终有 output；如果没有，需要明确 fake patch 或 pending。
- trim/custom/agent_end 前要 flush tool call，避免消息顺序破坏。

## Related Implementations

- 外部项目如果只保存 messages，不算覆盖这个概念。
- 值得关注的是能否提供更好的 trace schema、replay、ledger invariant 校验。
