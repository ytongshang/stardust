---
title: Risk Control
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - guardrails
  - risk
---

# Risk Control

Risk Control 是产品/安全侧的内容风控、资源风控、额度风控和异常阻断机制。

## 它解决什么问题

Agent harness 不只要“能完成任务”，还要控制内容风险、资源风险、成本风险和异常行为。Risk Control 是比单个 permission 更高层的风险判断，用来决定是否允许、降级、阻断或要求人工介入。

它需要覆盖：

- content risk：不安全内容、合规要求、敏感信息。
- resource risk：高成本调用、批量操作、危险文件修改。
- behavior risk：循环、异常 tool pattern、prompt injection。
- account risk：rate limit、quota、abuse detection。
- operational risk：外部服务异常、系统过载、灰度策略。

## 常见做法

常见机制包括：

- risk scoring：给 action、request、tool call 或 run 打风险分。
- policy gate：allow、deny、degrade、ask approval。
- quota control：token、tool calls、parallel runs、external API usage。
- anomaly detection：异常重试、异常输出、危险路径访问。
- incident trace：把风险决策写入 trace 和 audit log。

## 边界

- [[harness/guardrails/permissions/index|Permissions]] 管授权边界；Risk Control 管更广的风险评估。
- [[harness/guardrails/approvals-hitl/index|Approvals & HITL]] 是高风险动作的处理方式之一。
- [[harness/observability/failure-taxonomy/index|Failure Taxonomy]] 可以记录风险阻断和异常类型。

## 目录

- [[harness/guardrails/risk-control/implementations/uxarts-agent|uxarts-agent 实现]]
