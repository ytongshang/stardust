---
title: uxarts-agent — Tool Permissions
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/prototype_tools_builder.py
  - uxarts-agent/uxarts/web/modules/getui/agent/biz/sub_designer/sub_agent_tools_builder.py
tags:
  - uxarts-agent
  - permissions
---

# uxarts-agent — Tool Permissions

uxarts-agent 的工具权限目前主要由工具裁剪和 editor policy 实现：

- plan mode 设置 editor write allowlist，只允许 `internal/plans/plan.md`。
- architecture planning 未确认时只允许写 README。
- consult 模式使用 readonly filesystem。
- sub-agent 删除 main-only 工具，例如 `SendMessage`、`TaskStop`、plugin 配置类工具。
- stage 非 4 时裁剪 Supabase data tools。

权限规则目前没有统一 policy engine，散落在 tools builder、TextEditor allowlist 和具体工具实现里。
