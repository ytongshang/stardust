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

## 目录

- [[harness/context/context-injection/uxarts-agent|uxarts-agent 实现]]

## 常见注入时机

- first turn
- every turn
- after compact
- sub-agent start

## 后续收集

外部项目如果显式建模 context injection timing、context block provider、cache-safe context 或 per-agent injection policy，可以在本目录下新增笔记。
