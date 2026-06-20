---
title: Compaction
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - compaction
  - context
---

# Compaction

Compaction 是 context window 不够时，对历史进行摘要、替换和恢复的机制。它包括触发、summary、trim item、恢复拼接和 post-compact context 注入。

## 目录

- [[harness/context/compaction/uxarts-agent|uxarts-agent 实现]]

## 子问题

- 什么时候触发压缩。
- 压缩前后 run ledger 如何变化。
- summary 和当前项目状态怎么拼回上下文。
- 不同 agent 类型是否重新注入 static context。
- 如何评估 summary 质量。

## 后续收集

外部项目如果有分层压缩、cache-safe compact、summary eval、trace replay 或 deterministic compression，可以在本目录下新增笔记。
