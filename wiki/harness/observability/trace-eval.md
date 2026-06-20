---
title: Trace & Eval
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/tracing
  - uxarts-agent/uxarts/web/modules/getui/task/getui_runner.py
tags:
  - agent-harness
  - observability
  - eval
---

# Trace & Eval

## 这个概念是什么

Trace 记录 agent run 中发生了什么。Eval 判断 harness 的某个组件是否让 agent 更稳定、更可恢复、更低成本或更容易完成任务。

对 harness 来说，eval 不应只评模型输出，也要评工具策略、上下文策略、压缩、sub-agent、pending/resume、workspace merge。

## uxarts-agent 现在做什么

观测来源：

- `RunItem` 事件流。
- `RunHooks` / `GetuiRunHooks`。
- `TraceContextData`。
- `agent_run_stats`。
- `TaskMonitor`。
- fake tool item 飞书上报。
- status message / skills message 后台任务。
- model token usage。

已有 eval 相关：

- `run_e2e_test` tool。
- tool args tests。
- sub-agent tests。
- plan mode tests。
- prompt builder / capability context tests。

## uxarts-agent 怎么做

每个重要事件都能落 item 或 hook：

- agent start。
- user input。
- model generation。
- tool call。
- tool output。
- handoff。
- trim。
- customize。

tool 执行时会带：

- tool_call_item_id。
- agent name/id。
- memory/context。
- session/message/user_message id。

## 不能丢的细节

- `RunItem` 已经接近 trace schema，不要另起一套孤立日志。
- fake item report 是 ledger invariant 破坏的观测入口。
- eval 应该能基于保存的 run_items replay。
- token usage 和 tool duration 应该进入 harness eval。

## 后续值得补

- `RunItem + RuntimeProfile` replay。
- 压缩前后 answer quality eval。
- tool profile ablation。
- sub-agent fanout/merge eval。
- lifecycle failure recovery eval。

## Related Implementations

- 外部项目值得关注：OpenTelemetry trace schema、workflow replay、agent benchmark harness、tool-use eval。
