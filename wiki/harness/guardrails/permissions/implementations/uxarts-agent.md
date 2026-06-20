---
title: uxarts-agent — Permissions
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/prototype_tools_builder.py
tags:
  - uxarts-agent
  - permissions
---

# uxarts-agent — Permissions

uxarts-agent 的权限主要通过工具可用性、editor allowlist 和具体工具实现共同控制。

例子：

- plan mode 只能写 `internal/plans/plan.md`。
- architecture planning 未确认时只允许写 README。
- consult 模式 filesystem readonly。
- sub-agent 删除 main-only tools。
- remote plugin tools 根据 stage/agent type 裁剪。

当前没有单独的 permission engine。
