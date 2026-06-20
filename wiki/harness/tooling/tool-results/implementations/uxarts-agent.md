---
title: uxarts-agent — Tool Results
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/types/tool.py
  - uxarts-agent/uxarts/core/tool/tool_helper.py
tags:
  - uxarts-agent
  - tool-results
---

# uxarts-agent — Tool Results

uxarts-agent 的工具结果用 `ToolResultWithMessages` 承载：

- `output`: 模型可读输出。
- `messages`: 可追加到上下文的 prompt messages。
- `attachments`: 保存到后端的附件。
- `is_error` / `error_type`: 错误语义。

执行后会生成 `UxToolCallOutputItem`，进入 event log，并通过 hooks 保存到 center。
