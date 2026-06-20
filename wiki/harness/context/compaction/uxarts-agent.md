---
title: Compaction
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/conversation_context_window.py
  - uxarts-agent/uxarts/web/modules/getui/task/getui_generate_task.py
  - uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus/AgentCompleteMessageListener.java
tags:
  - agent-harness
  - compaction
  - context
---

# Compaction

## 这个概念是什么

Compaction 是 context window 不够时，对历史进行摘要、替换和恢复的机制。它不是简单 summarization，而是一整套 lifecycle：

- 判断是否需要压缩。
- 标记哪一轮需要压缩。
- 生成 summary。
- 写入 trim item。
- 下一轮或当前轮把 summary 和必要状态重新拼进上下文。

## uxarts-agent 现在做什么

现有压缩分两类：

- cross-turn compact：一轮结束后 center 标记并触发 summary，下轮使用 summary。
- within-turn compact：当前 run 中创建新的 user message，继续执行。

压缩后上下文不是只有 summary，还会拼：

- previous conversation summary。
- current project structure。
- post-compact static context。
- 原始当前 user message 或 continue prompt。

## uxarts-agent 怎么做

触发：

```text
GetuiGenerateTask.run
  -> check_has_exceed_conversation_context_threshold(token_usage)
  -> completeMessage(needCompress=True)
  -> AgentCompleteMessageListener.MarkCompressStatus
  -> doCompress
```

恢复：

```text
handle_compact_new_user_content_list
  -> _generate_conversation_summary_text
       previous-conversation-summary
       current_project_structure
  -> _get_post_compact_static_blocks
  -> append original current user content
```

当前轮继续：

```text
handle_create_new_user_content_list
  -> summary text
  -> post-compact static blocks
  -> system-generated continue prompt
```

post-compact static context：

- main agent: `build_static_context(context)`。
- general-purpose sub-agent: `general_purpose_static_context(context)`。
- read-only sub-agent: usually nothing beyond summary。

## 压缩前后上下文怎么拼

压缩前：

```text
historical run_items
  -> MessageLoader
  -> AgentRunner.get_chat_messages
  -> model input
```

压缩后下一轮：

```text
summary block
current_project_structure block
post_compact_static_blocks
current user content
```

压缩后当前轮继续：

```text
summary block
current_project_structure block
post_compact_static_blocks
"continue where you left off" system-reminder
```

## 不能丢的细节

- summary 本身永远保留。
- static context 是否重新注入由 agent type 决定。
- 项目结构会随 summary 一起注入，帮助模型恢复 workspace state。
- compact 后不要因为 `compressed=True` 在后续每一轮重复当作 first turn。
- `.superun/memory/` wiki memory 是 post-compact static context 的一部分。

## Related Implementations

- 外部实现如果只有“超过 token 就 summarise”，不算完整。
- 值得记录的是更好的 summary quality eval、trace replay、cache-safe compact、分层压缩策略。
