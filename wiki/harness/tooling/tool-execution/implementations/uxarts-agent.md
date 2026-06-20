---
title: Tool System
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/types/tool.py
  - uxarts-agent/uxarts/core/agent/tools/tool_runner.py
tags:
  - agent-harness
  - tools
---

# Tool System

## 这个概念是什么

Tool System 是模型可行动能力的执行层。它包括 schema、参数解析、错误回传、并发控制、执行顺序、挂起和结果消息。

## uxarts-agent 现在做什么

核心类型：

- `FunctionTool`
- `AgentTool`
- `ToolResultWithMessages`
- `ToolInterrupted`
- `ToolRunFunction`
- `ToolRunUnknown`
- `ToolRunHandoff`

工具可以返回：

- 普通 output。
- 多条 prompt messages。
- attachments。
- error result。
- interrupted signal。

## uxarts-agent 怎么做

参数处理：

- `json.loads`。
- Pydantic validation。
- nested JSON-like string 修复。
- control chars escape 修复。
- `json_repair` fallback。
- validation retry。

错误处理：

- tool error 会转成模型可读的 tool result。
- 部分异常直接 raise，例如用户中断、余额不足、session 不存在。
- unknown tool 返回 tool result，不直接 crash。

执行：

- 普通工具并行。
- 带 `sequence_priority` 的工具顺序执行。
- interrupted 工具只保存 call，等待外部回复。
- `ToolRunner` 防同一轮并发写同一文件。

## 不能丢的细节

- tool args repair 是生产关键能力。
- tool output 不只是字符串，也可能带 messages 和 attachments。
- interrupted tool 是 HITL/pending/resume 的底座。
- sequential priority 与 parallel execution 要共存。
- unknown tool 必须可恢复地反馈给模型。

## Related Implementations

- 外部项目值得关注：tool schema 压缩、tool error taxonomy、工具权限元数据、tool call replay。
