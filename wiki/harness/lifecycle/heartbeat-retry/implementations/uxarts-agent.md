---
title: uxarts-agent — Heartbeat & Retry
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/heartbeat_manager.py
  - uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentApplicationCommandService.java
tags:
  - uxarts-agent
  - heartbeat
---

# uxarts-agent — Heartbeat & Retry

Python `GetuiGenerateTask` 启动 heartbeat，center 更新 message/session heartbeat。

retry 相关：

- `doRun` 调用前检查 message `needRetry()`。
- 失败 complete 如果可重试，会 leave running queue 并进 delay queue。
- 重试调用模型前会 `fillIncompleteToolCallOutput`，补齐缺失工具输出。

heartbeat 既是运行状态，也是 retry 判断依据。
