---
title: uxarts-agent — Tool Definition
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/types/tool.py
tags:
  - uxarts-agent
  - tools
---

# uxarts-agent — Tool Definition

uxarts-agent 用 `FunctionTool` / `AgentTool` 描述工具。`function_tool()` 可以把普通 Python 函数包装成工具，生成参数 schema、描述、参数验证和调用入口。

关键点：

- Pydantic model 作为参数结构。
- JSON schema 暴露给模型。
- 工具可带 `failure_error_function`、`sequence_priority` 等执行元数据。
- tool definition 和实际工具集合裁剪分开处理。
