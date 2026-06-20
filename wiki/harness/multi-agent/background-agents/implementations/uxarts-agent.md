---
title: uxarts-agent — Background Agents
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/async_sub_agent/sub_agent_run_registry.py
  - uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/SubAsyncAgentApplicationService.java
tags:
  - uxarts-agent
  - background-agents
---

# uxarts-agent — Background Agents

uxarts-agent 的 background agent 是独立 mode 13 `GetuiGenerateTask`：

- 独立 session/context/workspace。
- 由 `SubAgentRunSpec` 决定 agent_name、user_message_transform、init_messages、post_compact_context_blocks、startup loader。
- center 保存 main/sub 关系和 pending tool id。
- 完成后由 center listener 回投父 session。

这不是进程内同步工具调用，而是完整 agent run。
