---
title: uxarts-agent — Cancellation
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentApplicationCommandService.java
tags:
  - uxarts-agent
  - cancellation
---

# uxarts-agent — Cancellation

uxa-center 的 `softStopRound` 是普通 stop 和 sub-agent stop 的公共核心：

- domain completeMessage success。
- 清理 session 级 queue key。
- 清理 round 级 `sessionId#replyMessageId` queue key。
- post `StopSessionEvent`。

之前修过一个关键问题：非首轮 stop 如果清错 round key，会泄漏全局并发名额。
