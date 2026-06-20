---
title: Tool Definition
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
---

# Tool Definition

Tool Definition 定义模型能看到和调用的工具接口，包括名称、描述、JSON schema、参数模型、返回约定和触发语义。

## 它解决什么问题

Tool Definition 是 model 和 runtime 之间的 contract。模型只能根据 definition 理解工具能做什么、参数怎么填、什么时候该调用；runtime 也只能根据 definition 做 validation、execution 和 result mapping。

它需要让几个边界清楚：

- tool 的语义边界是什么，不是什么。
- 参数 schema 如何表达必填、枚举、默认值和复杂结构。
- tool call 的返回会以什么形式进入下一轮 context。
- tool 是否有副作用、权限要求、成本或等待用户确认。
- 不同 model provider 的 tool schema 差异如何适配。

## 常见做法

常见实现会把 tool definition 拆成几层：

- public schema：暴露给模型的 name、description、parameters。
- runtime handler：真实执行函数和执行上下文。
- permission metadata：资源访问、危险操作、approval policy。
- result schema：成功、失败、partial result、artifact refs 的返回约定。
- adapter：把内部 tool spec 渲染成 OpenAI tools、Anthropic tools、MCP tools 等格式。

好的 definition 不只是“函数签名”，还会通过描述和 schema 降低误调用率，并让 tool runner 能做一致的 validation 和 error recovery。

## 边界

- [[harness/tooling/tool-availability/index|Tool Availability]] 决定本轮暴露哪些 tools；Tool Definition 决定每个 tool 是什么。
- [[harness/tooling/tool-execution/index|Tool Execution]] 负责运行 handler；Definition 只定义 contract。
- [[harness/tooling/tool-permissions/index|Tool Permissions]] 可以附着在 definition 上，但权限判定通常是独立 policy。

## 目录

- [[harness/tooling/tool-definition/implementations/uxarts-agent|uxarts-agent 实现]]
