---
title: Agent Communication
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - multi-agent
---

# Agent Communication

Agent Communication 处理 agent 之间如何发消息、请求信息、回传结果和继续执行。

## 它解决什么问题

多 agent 不只是创建多个 run，还需要通信协议。Agent Communication 定义 agent 之间如何请求、回答、等待、通知和传递 structured result，避免所有协作都退化成自由文本聊天。

它需要回答：

- message 的 sender、receiver、thread/run relation 如何表示。
- 请求是否需要 response，response 的 schema 是什么。
- agent 间是否允许直接读写彼此 workspace/memory。
- communication 是否同步、异步、streaming 或 callback。
- 消息如何进入 Event Log、trace 和 user-visible UI。

## 常见做法

常见机制包括：

- task message：parent 给 child 的任务说明和约束。
- ask/answer：child 向 parent 或 user 请求补充信息。
- result callback：child 完成后回传 summary、artifacts、status。
- broadcast / fanout：coordinator 同时通知多个 agents。
- protocol envelope：message type、correlation id、deadline、priority。

## 边界

- [[harness/multi-agent/coordination/index|Coordination]] 决定谁做什么；Agent Communication 决定他们怎么交换信息。
- [[harness/runtime/event-log/index|Event Log]] 记录通信事实，但不定义通信协议。
- [[harness/lifecycle/pending-resume/index|Pending & Resume]] 常用于等待通信结果。

## 目录

- [[harness/multi-agent/agent-communication/implementations/uxarts-agent|uxarts-agent 实现]]
