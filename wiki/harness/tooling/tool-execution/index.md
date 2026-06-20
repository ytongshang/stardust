---
title: Tool Execution
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
---

# Tool Execution

Tool Execution 负责实际运行模型请求的工具调用，包括并发、顺序、执行上下文、超时、异常传播和结果保存。

## 它解决什么问题

模型发出 tool call 只是意图，真正执行需要 runtime 处理 validation、permission、concurrency、timeout、side effects、error mapping 和 result persistence。Tool Execution 把这些工程细节收在一个可控 runner 里。

它需要回答：

- tool call 是否合法，参数是否能 parse / repair。
- tool 是否可以并发执行，还是必须顺序执行。
- 执行时能访问哪些 workspace、credentials、user/session state。
- tool hang、timeout、exception、partial failure 如何回传给模型。
- 执行结果写入 Event Log、artifact、trace 还是只作为 observation。

## 常见做法

常见 tool runner 会包含：

- preflight：schema validation、permission check、approval check。
- dispatch：按 tool name 找 handler，注入 execution context。
- scheduling：parallel / sequential / queued execution。
- isolation：sandbox、subprocess、remote executor、transaction boundary。
- result mapping：把 return value、exception、artifact refs 包成 Tool Results。

## 边界

- [[harness/tooling/tool-definition/index|Tool Definition]] 是 contract；Tool Execution 是 contract 的运行时实现。
- [[harness/tooling/tool-error-recovery/index|Tool Error Recovery]] 处理失败后给模型的可恢复反馈。
- [[harness/guardrails/permissions/index|Permissions]] 和 [[harness/guardrails/sandbox-policy/index|Sandbox Policy]] 会限制 execution，但不是 runner 本身。

## 目录

- [[harness/tooling/tool-execution/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部项目如果有 tool runner、parallel tool execution、tool sandbox、tool scheduling 或 execution replay，可以放到本目录。
