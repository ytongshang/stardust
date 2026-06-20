---
title: uxarts-agent — Teams
type: implementation
created: 2026-06-20
updated: 2026-06-20
tags:
  - uxarts-agent
  - teams
---

# uxarts-agent — Teams

uxarts-agent 当前没有独立的 team abstraction。已有能力更接近：

- 主 agent + 多个 background sub-agent。
- 子 agent type 代表角色能力，例如 general-purpose、explore、codex-review、superun-guide。
- task manager 记录 owner 和任务状态。

如果后续需要 team 概念，可以在 sub-agent 类型、任务依赖和协调协议之上抽象。
