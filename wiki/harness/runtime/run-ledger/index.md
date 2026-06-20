---
title: Run Ledger
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - event-log
---

# Run Ledger

Run Ledger 是 agent run 的事实日志。它比普通 messages 更细，记录 agent start、user input、model generation、tool call/output、handoff、trim、custom event 等。

## 目录

- [[harness/runtime/run-ledger/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部项目如果有更好的 event schema、trace replay、ledger invariant 校验或恢复机制，可以在本目录下新增实现笔记。
