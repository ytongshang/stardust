---
title: Subagents
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/async_sub_agent/sub_agent_run_registry.py
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/subagent
  - uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/SubAsyncAgentApplicationService.java
tags:
  - agent-harness
  - multi-agent
---

# Subagents

## 这个概念是什么

Subagent 是主 agent 把部分任务委托给另一个 agent 的机制。它可以是同步 inline 工具，也可以是独立 background session。

## uxarts-agent 现在做什么

两类：

- inline subagent: 作为工具同步执行，返回结果。
- background subagent: 独立 mode 13 `GetuiGenerateTask`，有自己的 session/context/workspace。

background sub-agent 支持：

- `SendMessage`: 主 agent 给子 agent 发消息。
- `AskMainAgent`: 子 agent 请求主 agent。
- `TaskStop`: 主 agent 停止子 agent。
- completion 回投。
- artifacts merge。
- fanout limit。
- 防递归。

## uxarts-agent 怎么做

`SubAgentRunSpec` 已经是一个局部 RunSpec：

- `agent_name`
- `user_message_transform`
- `init_messages`
- `post_compact_context_blocks`
- `startup_resource_loader`

general-purpose sub-agent：

- main-like。
- 可写 workspace。
- 注入 static/per-turn context。
- 压缩后重新注入 static context。

read-only sub-agent：

- lightweight user message。
- 不注入 heavy static context。
- 不需要 workspace init framing。

center 侧：

- 记录 parent/main/sub 关系。
- 管 pending tool id。
- 处理 SendMessage 是 append 还是 batchReplyToolCall。
- 子完成后回投父 session。

## 不能丢的细节

- background subagent 是完整独立 run，不是进程内 function call。
- 子 agent plugin state 查 main session。
- 子 agent 不应随意再开 background agent。
- `AskMainAgent` 走 interrupted tool + center pending/resume。
- launch idempotency 依赖 tool_call_item_id / sub index。
- merge artifacts 要处理冲突，而不是直接覆盖。

## Related Implementations

- 外部项目如果只有同步 `task()`，不算覆盖 background sub-agent。
- 值得关注的是 sub-agent lifecycle、conflict merge、multi-agent trace、子任务 eval。
