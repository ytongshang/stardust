---
title: AgentSpace Runtime Brokerage
type: implementation
created: 2026-06-23
updated: 2026-06-23
sources:
  - https://github.com/HKUDS/AgentSpace
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/daemon/src/provider-runtime.ts
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/daemon/src/agent-router/router.ts
tags:
  - agent-harness
  - runtime
  - implementation
---

# AgentSpace Runtime Brokerage

AgentSpace 最值得记录的点，不是它自己又写了一个新的 coding agent loop，而是它把 Claude Code、Codex、OpenClaw、Hermes 等 external runtimes 提升成一层可注册、可路由、可诊断、可授权的 execution substrate。

## 它做了什么

`packages/daemon/src/remote-daemon.ts` 启动时先 `detectProviders()`，再把当前机器上可执行的 provider CLIs 注册成多个 runtimes，并通过 `register / heartbeat / claimTask / reportMessages / uploadOutputBundle` 这组 daemon API 持续和平台同步。

`packages/daemon/src/provider-runtime.ts` 决定具体 task 交给哪个 provider path 执行：

- `codex`、`claude`、`openclaw`、`hermes` 走统一的 `runAgentRouterProviderTask(...)`。
- `gemini`、`opencode`、`nanobot` 目前仍保留 provider-specific path。
- 每个 runtime 带有 provider、可执行路径、模式、健康信息、profile/model metadata。

`packages/daemon/src/agent-router/router.ts` 则提供更薄的一层 runtime adapter：

- detect。
- buildLaunch。
- run。
- normalizeError。

也就是说，AgentSpace 的核心不是“让所有 runtime 共享一个 loop”，而是让它们共享一层 launch / event / diagnostics / fallback contract。

## 它为什么和常见做法不同

很多系统只是做 provider switch：换个模型名、换个 API endpoint。AgentSpace 做的是 runtime switch：每个目标本身就是一个完整 CLI agent runtime，有自己的 session 语义、工具模型、输出事件和失败模式。

它最重要的设计选择是：

- 平台自己维护 runtime catalog，而不是把 provider 视为透明后端。
- continuity 的事实源不完全托付给 provider session。
- runtime capability 在 launch 前先做一轮真实诊断。
- failure 先标准化，再决定 retry / fallback / surfacing。

## 连续性模型

`task-context.ts` 的 `buildRouterSessionContextLines(...)` 明确告诉模型：platform-level router session 才是 continuity source，provider-native session 只是“可复用时就复用”的 resume hint。

这和 `provider-runtime.ts` 中的恢复策略是一致的：

- 如果某个 runtime 的 provider session 可继续，就把 session id 传给 adapter。
- 如果检测到 session 缺失，例如 Codex rollout 丢失、Claude conversation 不存在、OpenClaw session missing，就发 `provider_session_invalid` 事件。
- 随后改为 `sessionId: undefined` 重新跑一次，由平台注入的 channel history、router transcript、document context、knowledge、runtime apps 来完成 cold rebuild。

这使它不会把“恢复”错误地等同为“provider 记得上次会话”。

## 失败与能力归一化

`agent-router/events.ts` 把各家 native events 映射成统一事件：

- `text_delta`
- `thought_delta`
- `tool_started`
- `tool_output`
- `tool_finished`
- `session_updated`

`provider-runtime.ts` 再把 diagnostics 继续映射成统一 provider error code / category。这样 UI、queue、audit 和 retry policy 面对的是统一语义，而不是每个 provider 一套 stderr 规则。

同时，`buildRuntimeToolCapabilities(...)` 会把三类能力合并：

- builtin output CLI；
- builtin Google Workspace CLI；
- CLI-Hub runtime apps；

再交给 `capabilities.ts` 做 diagnostic 和 allowed shell pattern 计算。也就是说，tool availability 先经过 runtime reality，再交给模型。

## 最值得借鉴的点

- 把 platform session 和 provider session 拆开，避免 continuity 完全依赖第三方 CLI。
- 把 runtime capability inventory 放到 launch 前，先诊断再执行。
- 用统一 runtime adapter 层收敛多家 CLI 的 launch / parse / normalizeError，而不是把这些逻辑散落在业务层。
