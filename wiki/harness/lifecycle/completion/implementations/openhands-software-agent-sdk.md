---
title: OpenHands Software Agent SDK — Completion
type: implementation
created: 2026-06-24
updated: 2026-06-24
sources:
  - https://github.com/OpenHands/software-agent-sdk
  - /Users/rancune/Work/agent/automations/agent-harness-radar/2026-06-22/OpenHands__software-agent-sdk/openhands-sdk/openhands/sdk/conversation/goal/controller.py
  - /Users/rancune/Work/agent/automations/agent-harness-radar/2026-06-22/OpenHands__software-agent-sdk/openhands-sdk/openhands/sdk/conversation/goal/runner.py
  - /Users/rancune/Work/agent/automations/agent-harness-radar/2026-06-22/OpenHands__software-agent-sdk/openhands-agent-server/openhands/agent_server/event_service.py
tags:
  - openhands
  - completion
  - goal-driver
---

# OpenHands Software Agent SDK — Completion

OpenHands Software Agent SDK 的 `/goal` 是一种 goal driver：它不替换 agent loop，也不是 workflow engine，而是在 agent 每次完成后从外层做验收。

## 核心想法

普通 `conversation.run()` 在 agent 进入 finished 状态后就结束。`/goal` 多加一层：

1. driver 把 objective 作为用户消息发给同一个 conversation。
2. agent 按原本的 loop、tools、critic 和 context 机制完成一次 run。
3. driver 调用 judge LLM 读取 conversation events，判断目标是否已有足够证据完成。
4. 如果 judge 认为证据不足，driver 把缺口整理成 follow-up user message，再启动下一次 run。
5. 如果 judge 认为完成，或达到 `max_iterations`，driver 才进入 terminal outcome。

这个分层很干净：agent 负责干活，judge 负责验收，driver 负责生命周期推进。它不规定中间步骤，所以不是 workflow engine；它也不嵌在单次 action 或 final response 里，所以不是普通 critic。

## 实现位置

- `GoalController` 只保存 objective、iteration、`max_iterations`，并在 `on_run_finished()` 中调用 `judge_goal()` 决定 `GoalContinue` 或 `GoalDone`。
- `run_goal()` 是同步 SDK driver：`send_message()`、`run()`、judge、必要时继续追加 follow-up。
- agent-server 的 `EventService.start_goal_loop()` 把同样的 controller 挂成 background task。
- `EventService._run_goal_loop()` 负责真正的 I/O：发送 objective/follow-up、等待 run task、发布 `ConversationStateUpdateEvent(key="goal")`。
- `EventService.stop_goal_loop()` 只取消 goal loop 本身，并写入 `interrupted` goal status。
- `EventService.resume_goal_loop()` 从最近一次持久化的 goal status event 里恢复 objective、iteration 和 max iteration。

## 和 completion 的关系

这里的关键不是“如何判断一次 agent step 是否结束”，而是“agent 自称完成后，系统是否接受这个完成”。`/goal` 把 completion 从 agent 内部的 finished signal 提升成外部 evidence gate。

这对 verifiable coding task 很有价值。例如：

- 测试必须真的跑过并通过。
- 文件必须存在且内容正确。
- 修复必须能从 trace、command output 或 artifact 中看到证据。

如果 agent 只说“已经完成”，但 event log 里没有足够证据，goal driver 会继续推进，而不是接受 final answer。

## 可借鉴点

- Completion 可以拆成两层：inner completion 是 agent loop 的终态，outer completion 是 evidence-based acceptance。
- 外层验收不需要接管 agent loop；用普通 user message 继续同一个 conversation 就能复用现有 context、tools、critic 和 event log。
- goal status 作为 `ConversationStateUpdateEvent` 持久化后，UI 可以显示进度，server restart 后也能恢复。
- `complete` 和 `capped` 被区分为不同 outcome，避免把 iteration cap 误认为成功。

## 边界

这种模式适合能从 workspace、tool output、test result 或 artifact 判断完成度的任务。不适合完全主观、开放式、没有明确证据标准的任务；这类任务里 judge 可能只是另一个偏好模型，不能提供真正更强的 completion guarantee。
