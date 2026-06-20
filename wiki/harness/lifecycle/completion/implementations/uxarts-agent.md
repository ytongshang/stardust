---
title: uxarts-agent — Completion
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/getui_generate_task.py
  - uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus/AgentCompleteMessageListener.java
tags:
  - uxarts-agent
  - completion
---

# uxarts-agent — Completion

Python 成功结束后调用 `completeMessage(success=True, needCompress=...)`。失败时映射 errorType/errorMessage/errorCanRetry。

center 完成后：

- 设置 END/EXCEPTION_END。
- leave running queue。
- post `CompleteMessageEvent`。
- 触发 git commit、chat queue drain、compress、wiki timer、sub-agent completion 回投等 listener。
- 成功 complete 可能受 `minDuration` 延迟。
