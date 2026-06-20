---
title: Agent Loop
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/agent/agent_runner.py
tags:
  - agent-harness
  - runtime
---

# Agent Loop

## 这个概念是什么

Agent loop 是一次 agent run 的核心执行循环：准备上下文，调用模型，解析模型输出，执行工具或 handoff，然后决定继续、结束、转交或挂起。

这属于通用 harness 概念，不是 GetUI 业务概念。任何 agent 产品只要支持工具、多轮、恢复、子 agent，都需要这层。

## uxarts-agent 现在做什么

`AgentRunner` 是通用执行器。它不直接理解 `mode/stage`，而是接收已经创建好的 `UxAgent`、历史 `RunItem`、`GetuiContext`、`RunHooks` 和 `RunConfig`。

它支持：

- 多 turn loop。
- streamed / non-streamed 模型调用。
- final output。
- tool call。
- unknown tool。
- handoff。
- flow transfer。
- interrupted tool。
- max turns / custom should_exit_loop。
- token usage 传递。
- memory hooks / tracing hooks。

## uxarts-agent 怎么做

核心流程：

```text
AgentRunner.run
  -> _prepare_for_first_run
  -> loop:
       _run_single_turn
       model.get_response / streamed_response
       _process_model_response
       execute_tools_and_side_effects
       decide NextStep
```

关键类型：

- `SingleStepRunInput`: 本轮 system、messages、delta_run_items、run_items、token_usage。
- `SingleStepResult`: 一次模型调用和工具执行后的结果。
- `NextStepFinalOutput`: 结束。
- `NextStepRunAgain`: 工具执行后继续。
- `NextStepHandoff`: 切换 agent。
- `NextStepTransfer`: flow transfer。
- `NextStepInterrupted`: 工具挂起，等待外部回复。

## 不能丢的细节

- `NextStepInterrupted` 不等于失败，它是正常的 pending/resume 入口。
- handoff 和 transfer 是两类概念：handoff 是 agent 间移交，transfer 更像流程结束后的下一 agent。
- `RunResult.next_run_items` 是下一轮重建上下文的基础。
- token usage 会向下一轮传递，后续压缩判断依赖它。

## Related Implementations

- 待补充：LangGraph / DeepAgents / OpenAI Agents SDK / Claude Code 等实现中，agent loop 与 graph/runtime 的边界差异。
