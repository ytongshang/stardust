---
title: uxarts-agent — Skill Discovery
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/plugin/capability_brief_injector.py
tags:
  - uxarts-agent
  - skills
---

# uxarts-agent — Skill Discovery

uxarts-agent 通过 capability context 暴露 skill 摘要：

- 每轮拉取 skill catalog。
- capability static context 给模型可用能力和 skill 摘要。
- 模型不需要一开始读完整 skill body。
- 某些 plugin/enableMode 可以触发相关 skill。

Skill discovery 依赖 `CapabilitySnapshot`，但概念上 skill 是独立能力包，不是 plugin state 本身。
