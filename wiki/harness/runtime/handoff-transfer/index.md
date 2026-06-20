---
title: Handoff & Transfer
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - runtime
  - handoff
---

# Handoff & Transfer

Handoff & Transfer 记录 agent 执行主体的切换：模型决定把任务交给另一个 agent、另一个流程或另一个 specialized worker。

## 它解决什么问题

单个 agent 不一定适合完成所有任务。Handoff & Transfer 让 runtime 能把任务交给更合适的执行主体，同时保留上下文、责任边界和结果回传路径。

它需要回答：

- transfer 的触发者是谁：model、tool、router、user 还是 system policy。
- 新 agent 拿到什么 input、history、workspace、memory 和 tool set。
- 原 agent 是等待结果、结束自己，还是继续并行执行。
- transfer 后的 output 如何回到原 run、产品层或 Event Log。
- 如果 transfer 失败、超时或被取消，责任如何回收。

## 常见做法

常见实现会把 transfer 建模为一个显式 runtime event，而不是只在 prompt 里描述“请交给别人”：

- 定义 target agent / worker / workflow。
- 构造 handoff payload：task、constraints、context summary、artifact refs。
- 保存 parent-child relation，方便 trace 和 result merge。
- 明确 return contract：final message、structured output、artifact diff、status。
- 给 transfer 设置 timeout、retry、permission 和 cancellation policy。

## 边界

- [[harness/multi-agent/subagents/index|Subagents]] 是一种常见 handoff 目标，但 Handoff & Transfer 也可以发生在 workflow、tool worker 或 hosted execution provider 之间。
- [[harness/multi-agent/agent-communication/index|Agent Communication]] 关注双方如何消息往来；Handoff & Transfer 关注执行主体和责任的切换。
- [[harness/workspace/workspace-merge/index|Workspace Merge]] 处理 transfer 后产物如何合并，不是 handoff 本身。

## 目录

- [[harness/runtime/handoff-transfer/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

关注外部项目如何定义 transfer semantics、handoff input/output、agent identity、handoff 后历史如何重建。
