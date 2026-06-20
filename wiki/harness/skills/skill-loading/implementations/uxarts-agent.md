---
title: uxarts-agent — Skill Loading
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/plugin/skill_tool.py
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/file_system/file_system_tool.py
tags:
  - uxarts-agent
  - skills
---

# uxarts-agent — Skill Loading

uxarts-agent 的 skill loading 采用 progressive disclosure：

- prompt/capability context 先给摘要。
- skill 文件包 materialize 到 `.superun/skills/<id>/`。
- 模型用普通 file tools 读取 `SKILL.md` 和相关资源。
- 对 manual/follow_plugin 未自动触发的 skill，可通过 Skill tool 显式安装。

这避免把所有 skill body 一次性塞进上下文。
