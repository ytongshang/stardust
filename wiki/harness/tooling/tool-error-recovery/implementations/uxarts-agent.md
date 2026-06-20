---
title: uxarts-agent — Tool Error Recovery
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/types/tool.py
  - uxarts-agent/uxarts/core/agent/agent_runner.py
tags:
  - uxarts-agent
  - tool-error-recovery
---

# uxarts-agent — Tool Error Recovery

uxarts-agent 在工具错误恢复上有几层处理：

- `tool_args_validate` 解析 JSON、修复嵌套 JSON-like string、escape 控制字符、使用 `json_repair` fallback。
- Pydantic validation 失败会转成模型可读错误。
- `failure_error_function` 把工具异常包装成 tool result。
- unknown tool 不直接 crash，而是形成 unknown tool result 反馈给模型。
- 用户中断、余额不足、session 不存在等异常会直接抛出，不伪装成普通工具错误。
