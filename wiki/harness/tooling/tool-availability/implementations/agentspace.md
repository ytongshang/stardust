---
title: AgentSpace Tool Availability
type: implementation
created: 2026-06-23
updated: 2026-06-23
sources:
  - https://github.com/HKUDS/AgentSpace
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/daemon/src/provider-runtime.ts
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/services/src/clihub/runtime-apps.ts
  - https://github.com/HKUDS/AgentSpace/blob/main/packages/services/src/skills/injection.ts
tags:
  - agent-harness
  - tooling
  - implementation
---

# AgentSpace Tool Availability

AgentSpace 在 Tool Availability 上最有意思的点，是它把 `skill` 和 `runtime app` 明确拆开：skill 只是 instructions materialization，runtime app 才是当前 runtime 真正可执行的 capability inventory。

## 它怎么分层

### Skill

`packages/services/src/skills/injection.ts` 会把 workspace skills 同时写到：

- `.agent_context/skills`
- provider-native 目录，例如 `.codex/skills`、`.claude/skills`

这解决的是“模型怎么读到 instructions”。

### Runtime App

`packages/services/src/clihub/runtime-apps.ts` 管的是另一层：

- 当前 runtime 已安装哪些 app。
- app 的 install/update/uninstall plan 是什么。
- readiness 是否通过。
- 哪些 app 可作为 task context 暴露。

这解决的是“当前机器上到底能执行什么”。

## 真正的 availability 发生在哪

`provider-runtime.ts` 的 `buildRuntimeToolCapabilities(...)` 合并三类来源：

- builtin `agent-space output` CLI；
- builtin Google Workspace CLI；
- CLI-Hub runtime apps；

随后 `agent-router/capabilities.ts` 再做：

- env/path 注入；
- allowed shell pattern 生成；
- diagnostic command 检查；
- denied/missing capability 归一化为统一 diagnostics。

也就是说，AgentSpace 的 tool availability 不是“把一堆 tool schema 给模型”，而是：

1. 先根据 runtime reality 确认什么软件真的存在。  
2. 再把存在且允许的命令翻译成模型可见 capability。  
3. skill 只作为使用说明，不自动等同于 capability。

## 为什么值得记

很多 agent systems 把 plugin、tool、skill、安装状态混成一层。AgentSpace 的拆分更清楚：

- skill answers “how to use”；
- runtime app answers “is it installed here”；
- tool capability answers “is it allowed for this task right now”。

这对 production coding agent 很重要，因为模型最容易在“会用”和“真能用”之间产生幻觉。
