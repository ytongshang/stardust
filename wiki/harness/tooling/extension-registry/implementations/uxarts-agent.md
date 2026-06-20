---
title: uxarts-agent — Extension Registry
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/plugin/capability_brief_injector.py
tags:
  - uxarts-agent
  - extensions
---

# uxarts-agent — Extension Registry

uxarts-agent 的扩展注册层主要是 capability/plugin 体系：

- `CapabilitySnapshot` 在 `GetuiContext` 生命周期内缓存 plugin catalog、skill catalog、plugin states。
- capability static context 给模型可用能力摘要。
- plugin enabled 后动态合并 remote manifest tools。
- sub-agent plugin 状态查 main session。

`CapabilitySnapshot` 是 uxarts-agent 的实现名，不应作为 wiki 的通用概念目录。
