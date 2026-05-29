---
title: "Prompt Caching — Compaction & Cache-Safe Forking"
type: concept
created: 2026-04-11
updated: 2026-04-11
sources:
  - https://x.com/trq212
tags:
  - AI/LLM/Caching
---

# Prompt Caching — Compaction & Cache-Safe Forking

上下文用完时的压缩操作（compaction）如何在不破坏缓存的情况下执行。

## 问题

朴素的 compaction 实现会导致全量 cache miss：

```
❌ 用独立 API 调用（不同 system prompt、无工具）做 compaction
   → 前缀与主对话完全不匹配，支付全额输入 token
```

## Cache-Safe Forking 方案

Compaction call 使用与主对话**完全相同**的：
- system prompt
- tool definitions
- 历史消息（作为前缀）

在末尾追加 compaction prompt 作为新 user message。API 视角下前缀几乎与主对话最后一次请求相同 → 缓存命中。

```
主对话最后一次请求：[system, tools, msg1, msg2, ..., msgN]
Cache-Safe Forking：[system, tools, msg1, msg2, ..., msgN, compaction_prompt]
                                                              ↑ 只多了这一条
```

## 注意事项

- 需要预留「compaction buffer」，确保有足够窗口空间容纳 compaction prompt 和输出 token
- 子任务分叉（subagent）同理：子 agent 尽量复用父 session 的 system prompt 和工具集

## 相关页面

- [[llm/concepts/prompt-caching/index]] — 概览
- [[llm/concepts/prompt-caching/system-design-principles]] — 系统架构层
- [[llm/concepts/prompt-caching/api-breakpoint-management]] — API 断点管理
