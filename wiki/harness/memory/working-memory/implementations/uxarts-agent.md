---
title: uxarts-agent — Working Memory
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/types/getui_context.py
tags:
  - uxarts-agent
  - working-memory
---

# uxarts-agent — Working Memory

uxarts-agent 的 working memory 主要存在于 `RunItem` 历史和 `GetuiContext` 运行态中。

包括：

- todos/features/thoughts/reasoning steps。
- task manager tasks。
- 当前 run 的 generated items。
- agent memory hooks / rebuilt memory。
- compact 前后的 summary 状态。

它主要服务当前 session/round，不等同于长期 wiki memory。
