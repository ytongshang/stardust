---
title: uxarts-agent — Tool Interruption
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/types/tool_mode.py
  - uxarts-agent/uxarts/core/agent/agent_runner.py
tags:
  - uxarts-agent
  - tool-interruption
---

# uxarts-agent — Tool Interruption

uxarts-agent 用 `ToolInterrupted` 表示工具需要挂起。

流程：

- 模型产生 interrupted tool call。
- `AgentRunner` 只保存 `UxToolCallItem`，不立即执行 output。
- 返回 `RunResultInterrupted`。
- center 将 message 置为 PENDING。
- 外部通过 `batchReplyToolCall` 写入 `UxToolCallOutputItem` 后续跑。

工具层只表达“调用被挂起”；生命周期层的 PENDING/RESUME 在 [[harness/lifecycle/pending-resume/index|Pending & Resume]]。
