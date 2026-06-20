---
title: uxarts-agent — Skill Versioning
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/plugin/capability_brief_injector.py
tags:
  - uxarts-agent
  - skill-versioning
---

# uxarts-agent — Skill Versioning

uxarts-agent 的 skill 版本信息主要来自 skill catalog。当前版本控制不是独立 harness 模块，而是和 capability snapshot / materialization 流程绑定。

已有语义：

- catalog 中有 skill id 和版本/状态信息。
- materialize 时按当前 catalog 内容写入 `.superun/skills`。
- 后续如果 skill 内容更新，需要处理 workspace 中旧文件的覆盖与缓存失效。

这块后续可以独立成更清晰的 skill package/version 管理。
