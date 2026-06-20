---
title: Center Orchestrator
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentApplicationCommandService.java
  - uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus/AgentCompleteMessageListener.java
tags:
  - agent-harness
  - lifecycle
---

# Center Orchestrator

## 这个概念是什么

Center Orchestrator 是产品级 agent run 的状态机。它负责排队、心跳、重试、挂起、恢复、完成、停止、队列 drain 和业务 listener。

这不是 Python agent loop 的一部分，但它是完整 harness 的必需层。

## uxarts-agent / uxa-center 现在做什么

核心流程：

```text
chat/retry
  -> doRun(sessionId, replyMessageId)
  -> heartbeat
  -> global queue
  -> fillIncompleteToolCallOutput
  -> doCallAgent
  -> Python GetuiGenerateTask
  -> heartbeat / saveItem / completeMessage
  -> complete listeners
```

关键状态：

- WAITING
- RUNNING
- PENDING
- END
- EXCEPTION_END

## uxa-center 怎么做

队列：

- 全局 running/waiting queue 用 `ZSetQueueHelper`。
- round key 使用 `sessionId#replyMessageId`。
- chat queue 处理同 session 连续输入和 merge。

心跳：

- Python 定时 heartbeat。
- center 更新 message/session heartbeat。
- `needRetry` 根据状态和 heartbeat 判断是否可重调度。

完成：

- `completeMessage` 可成功或失败。
- 失败且可重试时进入 delay queue。
- 成功时可能受 `minDuration` 延迟。
- 完成后 leave session key 和 round key。
- post `CompleteMessageEvent`。

pending/resume：

- interrupted tool 不 complete。
- center 设置 PENDING。
- 外部 `batchReplyToolCall` 写 tool output。
- 再进入 RUNNING/doRun。

stop：

- `softStopRound` complete 成功并清理 queue。
- 普通 stop 和 sub-agent stop 共用逻辑。

## 不能丢的细节

- `fillIncompleteToolCallOutput` 在重试调用模型前执行。
- session key 和 round key 都要清理，否则会泄漏并发名额。
- complete listener 会触发 git commit、chat queue drain、compress、sub-agent completion 等。
- sub session 的 queue drain 路径和 main session 不同，避免子完成回投竞态。
- minDuration 是产品体验/业务策略，但实现上属于 lifecycle policy。

## Related Implementations

- 纯 SDK 项目通常不覆盖这层。
- 值得关注的是 durable execution、workflow engine、lease/heartbeat、resume semantics、exactly-once completion。
