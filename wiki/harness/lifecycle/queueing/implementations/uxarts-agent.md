---
title: uxarts-agent — Queueing
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentApplicationCommandService.java
  - uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentQueueApplicationService.java
tags:
  - uxarts-agent
  - queueing
---

# uxarts-agent — Queueing

uxa-center 有两层队列：

- 全局 running/waiting queue：`ZSetQueueHelper` 控制并发。
- 会话 chat queue：处理同 session 连续输入、merge、drain。

round 级 key 使用 `sessionId#replyMessageId`，stop/complete/retry 时必须同时清理 session key 和 round key。
