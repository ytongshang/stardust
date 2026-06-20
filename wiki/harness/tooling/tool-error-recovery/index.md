---
title: Tool Error Recovery
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
  - recovery
---

# Tool Error Recovery

Tool Error Recovery 处理工具调用失败后的可恢复反馈，包括参数修复、unknown tool、异常归一化和让模型自我纠错的错误消息。

## 它解决什么问题

Tool call 失败很常见：模型给错参数、调用不存在的 tool、权限不足、外部服务超时、文件不存在、sandbox 报错。Tool Error Recovery 的目标不是隐藏错误，而是把错误变成模型和 runtime 都能继续处理的状态。

它需要回答：

- 哪些错误可以让模型修正后重试。
- 哪些错误应该直接停止、请求 approval 或转为 pending。
- 错误消息给模型看多少，避免泄漏敏感信息或制造噪音。
- 参数 parse 失败时是否尝试 JSON repair / schema repair。
- retry 是否有上限，如何避免 loop 卡死。

## 常见做法

常见机制包括：

- error normalization：把 exception、HTTP error、validation error 归一成有限 error type。
- repair hint：给模型可执行的参数修正提示，而不是堆栈信息。
- retry policy：按 tool、error type、attempt count 控制重试。
- fallback path：unknown tool、deprecated tool 或 unavailable tool 时给替代建议。
- trace tagging：把失败原因写入 trace 和 Failure Taxonomy。

## 边界

- [[harness/tooling/tool-results/index|Tool Results]] 承载 error envelope；Tool Error Recovery 决定错误如何反馈和是否重试。
- [[harness/tooling/tool-availability/index|Tool Availability]] 可以减少 unavailable tool 调用，但不能替代 recovery。
- [[harness/observability/failure-taxonomy/index|Failure Taxonomy]] 给错误分类命名，Recovery 使用这些分类做决策。

## 目录

- [[harness/tooling/tool-error-recovery/implementations/uxarts-agent|uxarts-agent 实现]]
