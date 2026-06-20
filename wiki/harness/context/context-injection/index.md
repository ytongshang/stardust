---
title: Context Injection
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - context
---

# Context Injection

Context Injection 决定哪些上下文在什么时机注入给模型。关键不是具体文案，而是注入时机、来源、缓存边界和是否随 agent 类型变化。

## 它解决什么问题

Agent 需要的 context 不只来自 user input，还来自 workspace、memory、retrieval、artifact、previous run、system state 和 external services。Context Injection 负责决定这些 context 何时进入模型，以及进入后是否应该持续、刷新或移除。

它需要控制：

- 注入时机：first turn、every turn、after compaction、after tool result、sub-agent start。
- 注入来源：workspace、memory、retrieval、product state、runtime state。
- 注入范围：只给当前 agent、给 child agent、给后续所有 turns，还是只给当前 turn。
- 注入形态：system block、developer block、user-adjacent content、tool metadata。
- 缓存边界：哪些 context 稳定可缓存，哪些每轮变化。

## 常见做法

好的实现通常会把 context 做成 provider 或 block：

- 每个 provider 声明 source、priority、refresh policy、budget cost 和 visibility。
- runtime 根据 agent profile、task state、workspace state 和 token budget 选择 context blocks。
- 注入结果记录到 Event Log 或 trace，便于解释模型为什么看到了某些信息。
- after compaction 时重新注入必要 static context，避免 summary 丢失关键约束。

## 目录

- [[harness/context/context-injection/implementations/uxarts-agent|uxarts-agent 实现]]

## 边界

- [[harness/context/input-transform/index|Input Transform]] 来自用户输入；Context Injection 是 runtime 主动补上下文。
- [[harness/context/prompt-assembly/index|Prompt Assembly]] 负责最终排序和渲染；Context Injection 负责选择和时机。
- [[harness/memory/memory-retrieval/index|Memory Retrieval]] 是一种 context source，不等于完整的 injection policy。

## 后续收集

外部项目如果显式建模 context injection timing、context block provider、cache-safe context 或 per-agent injection policy，可以在本目录下新增笔记。
