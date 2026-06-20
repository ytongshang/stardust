---
title: uxarts-agent — Context Budgeting
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/getui_generate_task.py
  - uxarts-agent/uxarts/core/prompt/utils/image_budget.py
tags:
  - uxarts-agent
  - context-budgeting
---

# uxarts-agent — Context Budgeting

uxarts-agent 的 context budgeting 目前主要体现在三处：

- `token_usage` 统计与 `check_has_exceed_conversation_context_threshold`，决定是否 `needCompress`。
- `tag_run_items_image_budget` 等图片预算处理，避免多模态内容过量。
- static/per-turn context 的注入时机控制，避免 capability/memory/wiki 在每次模型调用中重复出现。

当前没有独立的预算分配器；token 阈值、图片预算、压缩触发和 prompt 注入策略分散在 runner、task 和 prompt 工具中。
