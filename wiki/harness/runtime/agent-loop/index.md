---
title: Agent Loop
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - runtime
---

# Agent Loop

Agent Loop 是 agent runtime 的核心执行循环：准备输入、调用模型、解析输出、执行工具、处理 handoff/transfer/interrupt，并决定继续还是结束。

## 目录

- [[harness/runtime/agent-loop/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部项目如果有不同的 loop/graph/runtime 边界，可以在本目录下新增实现笔记。重点看它如何处理多 turn、工具执行、挂起恢复、handoff、max turn、streaming 和 failure recovery。
