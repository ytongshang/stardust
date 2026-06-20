---
title: Remote Tools
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
  - remote-tools
---

# Remote Tools

Remote Tools 是由外部服务、插件、MCP server 或 hosted execution provider 暴露给 agent 的工具。

## 它解决什么问题

不是所有 tool 都在本进程里。Remote Tools 让 agent 可以调用外部系统能力，例如 MCP server、plugin、browser service、cloud sandbox、database、internal API 或 third-party integration。

它需要处理：

- remote manifest / schema 如何发现和缓存。
- auth、credentials、tenant、rate limit 如何传递。
- remote latency、timeout、retry、partial failure 如何反馈。
- remote tool result 如何映射成本地 Tool Results。
- 外部能力变化时如何刷新 availability 和 permission。

## 常见做法

常见实现会有一个 adapter 层：

- discovery：读取 MCP manifest、plugin manifest、OpenAPI spec 或自定义 registry。
- schema mapping：把 remote tool schema 转成内部 Tool Definition。
- execution proxy：统一处理 auth、request、timeout、streaming、error normalization。
- lifecycle hook：remote tool 不可用时降级、隐藏或转为 pending。
- observability：记录 remote endpoint、duration、status、cost 和 trace id。

## 边界

- [[harness/tooling/extension-registry/index|Extension Registry]] 管理 remote capabilities 的目录；Remote Tools 关注调用语义。
- [[harness/tooling/tool-definition/index|Tool Definition]] 是统一 contract，Remote Tools 需要映射到这个 contract。
- [[harness/guardrails/permissions/index|Permissions]] 决定 remote credentials 和 external access 是否允许。

## 目录

- [[harness/tooling/remote-tools/implementations/uxarts-agent|uxarts-agent 实现]]
