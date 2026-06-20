---
title: uxarts-agent — Replay
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/types/items.py
tags:
  - uxarts-agent
  - replay
---

# uxarts-agent — Replay

uxarts-agent 尚未有完整 trace replay 系统，但已有基础：

- `RunItem` event log。
- model/tool/user/handoff/trim/custom item。
- center 持久化 item。
- `MessageLoader` 能把历史 item 解析回运行对象。

后续如果记录 RuntimeProfile/model/tool versions，可以进一步支持 replay。
