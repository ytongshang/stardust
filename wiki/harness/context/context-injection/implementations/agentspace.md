---
title: AgentSpace Context Injection
type: implementation
created: 2026-06-23
updated: 2026-06-23
sources:
  - https://github.com/HKUDS/AgentSpace
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/daemon/src/task-context.ts
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/daemon/src/channel-documents.ts
tags:
  - agent-harness
  - context
  - implementation
---

# AgentSpace Context Injection

AgentSpace 的 Context Injection 很 productized，但里面有一个很值得记录的 pattern：它把“平台恢复连续性所需的上下文”做成显式 injection blocks，而不是默认相信 provider session 还记得之前发生过什么。

## 属于哪个格子

- timing：task start / mention dispatch / cold rebuild resume。
- source：workspace state、router session、channel history、documents、knowledge、runtime apps、notifications。
- freshness：多数是 live state；router transcript 和 permission request 是近实时块。
- visibility：当前被派发的 agent runtime。
- shape：长文本 prompt block，加上落地到 workdir 的目录型 context。
- cache boundary：provider session 不可靠时，平台块必须可重放。

## 它注入了什么

`prepareDaemonTaskContext(...)` 会先收集并物化：

- attachments；
- assigned skills；
- knowledge pages；
- channel documents；
- runtime apps；
- unread notifications；
- document permission requests；
- router session context；

然后 `buildTaskPromptWithDocumentContexts(...)` 把这些来源分别渲染成 prompt sections，而不是混成一个大段说明。

比较特别的是三类块：

### 1. Router Session Context

`buildRouterSessionContextLines(...)` 会注入：

- `routerSessionId`
- `conversationKey`
- `continuationMode`
- `selectedRuntimeId`
- `previousRuntimeId`
- `providerSessionId`
- `fallbackReason`
- memory summary
- compact transcript / event log

而且直接告诉模型：如果 provider session 丢失或 runtime 切换，不要假设隐藏会话状态仍存在，应基于平台状态和可见 artifacts 继续。

### 2. Runtime App Context

`buildRuntimeAppContextLines(...)` 不把 skill 当成“能力存在”的证据。它只把 runtime 已安装、已启用的 CLI-Hub runtime apps 作为真实 capability inventory 注入，并明确写出：

- entry point
- version
- category
- requiresText
- 关联 SKILL.md

这相当于把“instructions available”和“software actually installed”拆成两个 context source。

### 3. Channel Document Context

`materializeChannelDocuments(...)` 会把每份授权文档落成 `document.md`、`blocks.json`、`meta.json`。  
`buildChannelDocumentPromptLines(...)` 再按 `viewer/editor/forwarder` 角色注入允许动作、Google Docs/Sheets data plane 规则，以及 output CLI 约束。

这里的重点不是文档产品本身，而是把“可编辑 artifact + 权限 + data plane contract”一起注入给 runtime。

## 为什么重要

很多系统会把 resume 设计成“继续同一个 provider session”。AgentSpace 则默认 provider session 只是 best-effort，加上这些平台级 context blocks 后，cold rebuild 仍然可工作。

这个模式对 production coding agent 很有价值：真正 durable 的不是 provider memory，而是平台可重放的上下文事实。
