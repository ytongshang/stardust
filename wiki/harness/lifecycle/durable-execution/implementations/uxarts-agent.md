---
title: uxarts-agent — Durable Execution
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent
  - uxa-center
tags:
  - uxarts-agent
  - durable-execution
---

# uxarts-agent — Durable Execution

uxarts-agent 的 durable execution 不是单个模块，而是多层组合：

- center 持久化 session/message/item。
- `RunItem` event log 可重建上下文。
- heartbeat/retry 检测丢失 run。
- pending/resume 处理外部输入。
- snapshots/artifacts 保存 workspace。
- complete/stop 清理生命周期状态。

这套更接近产品状态机，而不是纯 SDK runtime。
