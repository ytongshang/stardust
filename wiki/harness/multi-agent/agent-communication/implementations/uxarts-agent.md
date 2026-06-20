---
title: uxarts-agent — Agent Communication
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/agent_message
  - uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus
tags:
  - uxarts-agent
  - agent-communication
---

# uxarts-agent — Agent Communication

uxarts-agent 的 agent communication 主要包括：

- `SendMessage`: 主 agent 给后台子 agent 发消息。
- `AskMainAgent`: 子 agent 请求主 agent，走 interrupted tool / pending resume。
- completion 回投：子 agent 完成后把结果写回父 session。
- stop message: 主 agent 停止子 agent。

通信语义和 center lifecycle 深度绑定。
