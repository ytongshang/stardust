---
title: Extension Registry
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
  - extensions
---

# Extension Registry

Extension Registry 管理可扩展能力的目录、状态和启用关系，例如 plugin catalog、remote tool manifest、capability snapshot、integration registry。

## 它解决什么问题

Agent harness 往往需要动态接入 plugin、MCP server、skill、integration 或 hosted tool。Extension Registry 负责知道“有哪些能力存在、当前是否启用、来自哪里、版本是什么、能暴露给谁”。

它需要回答：

- capability 如何发现、安装、启用、禁用和更新。
- registry 中的能力如何映射成 tools、skills、context providers 或 UI actions。
- extension 的状态如何影响 Tool Availability 和 Skill Discovery。
- 版本变化、manifest 变化、权限变化如何被缓存和失效。
- 外部能力失败时如何降级和记录。

## 常见做法

常见 registry 会保存：

- extension id、source、version、manifest、capabilities。
- enabled state、permission grants、workspace/project scope。
- health status、last refresh time、error state。
- capability snapshot：当前 run 可见的 tools、resources、prompts 或 skills。
- update policy：自动刷新、手动刷新、pin version 或 cache busting。

## 边界

- [[harness/tooling/remote-tools/index|Remote Tools]] 是 registry 中一种能力的调用形态。
- [[harness/skills/skill-discovery/index|Skill Discovery]] 可以从 registry 读取 skill catalog。
- Extension Registry 管目录和状态，不负责具体 tool execution。

## 目录

- [[harness/tooling/extension-registry/implementations/uxarts-agent|uxarts-agent 实现]]
