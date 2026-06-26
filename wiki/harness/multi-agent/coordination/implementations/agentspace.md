---
title: AgentSpace Coordination
type: implementation
created: 2026-06-23
updated: 2026-06-23
sources:
  - https://github.com/HKUDS/AgentSpace
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/services/src/messages/messages.ts
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/domain/src/mention-plan.ts
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/services/src/shared/conversation-execution-workspaces.ts
tags:
  - agent-harness
  - multi-agent
  - implementation
---

# AgentSpace Coordination

AgentSpace 的多 agent coordination 不是 planner graph，也不是 abstract team runtime。它更像“把群聊里的 @Agent、handoff document 和执行工作区做成真实可治理的协作协议”。

## 它怎么启动协作

人类在 channel 里发消息后，`sendChannelHumanMessageSync(...)` 会：

- parse agent/human mentions；
- 检查 out-of-channel mention；
- 用 `parseMentionPlan(...)` 判断这是 parallel 还是 sequential；
- 如果是串行 handoff，就创建 `ChannelDocumentRun` 和多步 `ChannelDocumentRunStep`；
- 如果只是普通 mention，就给每个目标 agent enqueue task。

也就是说，coordination 的入口不是一个隐藏 planner，而是用户消息里的显式协作意图。

## 串行 handoff 的实现

`mention-plan.ts` 会根据 `然后 / 再 / 完成后 / 交给 @...` 这类标记，推断 step dependency 和 handoff kind。  
`messages.ts` 在顺序明确时创建 document run，并把 ready step 入队；不明确时拒绝猜测顺序，而是要求用户改写。

这个策略很实用：它没有追求“自动理解所有 multi-agent 语义”，而是把 ambiguity 当成 coordination failure，强制用户把依赖说清楚。

## Conversation Execution Workspace

`conversation-execution-workspaces.ts` 维护 `(conversation kind, channel, agent)` 维度的 execution workspace state，保存：

- `sessionId`
- `workDir`
- `lastTaskQueueId`
- `lastError`
- `autoContinuation`

这样后续同一个频道再次 @ 同一 agent，或 agent-output mention 触发下游 agent 时，都可以复用已有 session/workdir，或者至少知道上次运行落在哪个 execution workspace。

这是一种很具体的 coordination substrate：不是共享 memory，而是共享“这条协作线程上次跑在哪个 runtime workspace”。

## 防循环和边界

`dispatchAgentOutputMentionsSync(...)` 对 agent-output mention 有多层硬限制：

- 每次回复最多 dispatch 几个下游 agent。
- mention cascade depth 上限。
- 同一个 root message 可派生的总任务上限。
- self-mention 不触发。
- 同 source message 对同一 agent 不重复派发。

同时还继承 requester 权限，检查目标 agent 和 runtime 是否对该 requester 可用。

这说明 AgentSpace 的 coordination 重点不是“多聪明地自动协作”，而是“让协作进入真实 channel/workspace/permission 边界后仍然不爆炸”。
