---
title: uxarts-agent — Prompt Assembly
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/getui_generate_task.py
  - uxarts-agent/uxarts/web/modules/getui/task/prototype
tags:
  - uxarts-agent
  - prompt
---

# uxarts-agent — Prompt Assembly

uxarts-agent 的 prompt assembly 主要散落在 agent creator、prototype prompt modules、user input creator 和 mode/stage 路由里。

主要组成：

- `UxAgent.instructions`: system prompt 动态生成入口。
- prototype prompt modules: role、goal、workflow、execution environment、tool usage policy、memory、language/tone、injection protection、code style。
- `AgentCreateParams`: 把 stage、session_extra、round_extra、parallel_index 等业务路由传给 agent/prompt/tools builder。
- `<system-reminder>`: 用于插入运行时约束、context 和业务提醒。

当前问题是 prompt profile 没有独立对象，mode/stage 对 prompt、tools、context 都有影响。后续可以把这些选择编译成只读 `RuntimeProfile`。
