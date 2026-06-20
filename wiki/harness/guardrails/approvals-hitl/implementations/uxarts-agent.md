---
title: uxarts-agent — Approvals & HITL
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent
  - uxa-center
tags:
  - uxarts-agent
  - hitl
---

# uxarts-agent — Approvals & HITL

uxarts-agent 主要通过 interrupted tool + center pending 机制实现 HITL。

用途：

- 人工确认。
- `AskMainAgent`。
- 外部审批或超时失败。

和普通 tool error 不同，HITL 是正常流程：tool call 保存后 run 挂起，外部补 tool output 后续跑。
