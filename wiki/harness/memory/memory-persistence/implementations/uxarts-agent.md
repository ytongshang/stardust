---
title: uxarts-agent — Memory Persistence
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/types/getui_context.py
  - uxarts-agent/uxarts/web/modules/getui/task/memory
tags:
  - uxarts-agent
  - memory-persistence
---

# uxarts-agent — Memory Persistence

uxarts-agent 的长期记忆通过 wiki session 和 workspace 文件体现：

- wiki memory 从 center 查询 attachments。
- Python 预写到 `.superun/memory/`。
- overlay 时会更新变化文件并清理已删除文件。
- memory wiki generate 模式负责 ingest/lint 类任务。

记忆最终仍走 product artifact/attachment backend，而不是单独 memory database。
