---
title: Tool Availability
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
---

# Tool Availability

Tool Availability 决定某一轮、某个 agent profile、某个 task state 下到底向模型暴露哪些工具。它包括 tool pruning、dynamic tool loading、profile-based tool selection 和按 permission 隐藏工具。

## 它解决什么问题

工具越多不一定越好。过大的 tool set 会增加模型选择成本、误调用概率、prompt token 成本和安全风险。Tool Availability 负责在“能力足够”和“选择面可控”之间做取舍。

它需要回答：

- 当前任务真正需要哪些 tools。
- 哪些 tools 因 permission、workspace、model capability 或 product config 不可用。
- 是否要 progressive exposure，而不是一次性把全部 tools 给模型。
- tool set 变化是否需要记录，方便 replay/eval。
- 隐藏 tool 时是否需要给模型说明替代路径。

## 常见做法

常见策略包括：

- static profile：某类 agent 固定拥有一组 tools。
- dynamic routing：根据 user request、retrieval、task state 选择 tools。
- progressive exposure：先给少量核心 tools，需要时再加载更多。
- permission filtering：按用户、workspace、approval state 隐藏危险 tools。
- token-aware pruning：按 tool description/schema 成本裁剪低相关 tools。

## 边界

- [[harness/tooling/tool-definition/index|Tool Definition]] 定义 tool；Tool Availability 决定本轮是否暴露它。
- [[harness/tooling/tool-permissions/index|Tool Permissions]] 是 availability 的输入之一，但 availability 还包括相关性和成本。
- [[harness/skills/skill-loading/index|Skill Loading]] 可能带来新的 tools，最终仍需要经过 availability policy。

## 目录

- [[harness/tooling/tool-availability/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部项目如果有 staged tool exposure、dynamic tool routing、tool profile eval 或工具集压缩策略，可以放到本目录。
