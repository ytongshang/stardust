---
title: uxarts-agent — Pending & Resume
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/agent_runner.py
  - uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentApplicationCommandService.java
tags:
  - uxarts-agent
  - pending-resume
---

# uxarts-agent — Pending & Resume

流程：

```text
ToolInterrupted
  -> RunResultInterrupted
  -> center message = PENDING
  -> batchReplyToolCall 写 tool output
  -> doRun 续跑
```

这个机制支撑人工确认、`AskMainAgent`、外部自动审批和超时失败。
