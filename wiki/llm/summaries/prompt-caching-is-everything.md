---
title: "Lessons from Building Claude Code: Prompt Caching Is Everything"
type: summary
created: 2026-03-25
updated: 2026-04-11
sources:
  - https://x.com/trq212
tags:
  - AI/LLM/Caching
origin_type: blog
author: "Thariq Shihipar"
rating: ⭐⭐⭐⭐⭐
---

# Lessons from Building Claude Code: Prompt Caching Is Everything

## 📌 核心观点

> Prompt caching 是让长时间运行 Agent 产品成为可能的基础设施。整个系统架构必须围绕「前缀缓存」这一约束来设计——顺序、工具集、模型、分叉操作，任何一个细节的偏差都会导致缓存失效，代价昂贵。

---

## 📋 六条核心实践

### 1. Prompt 布局顺序

静态内容在前，动态内容在后：

```
① Base System Prompt & Tools    ← globally cached（跨所有 session）
② CLAUDE.md / 项目上下文        ← cached per project
③ Session Context               ← cached per session
④ Conversation Messages         ← 每轮动态增长
```

### 2. 用 Messages 传递动态信息

不要修改 system prompt（会导致 cache miss），而是在下一轮 user message 或 tool result 中插入 `<system-reminder>` 标签。

### 3. 不要中途切换模型

Prompt cache 与模型绑定。会话中途切换模型不仅不节省，反而因为需要重建缓存而更贵。正确做法：用 subagent 完成模型切换。

### 4. 不要中途增减工具

工具定义属于缓存前缀，中途增删会使整个会话缓存失效。

**Plan Mode 的正确实现**：始终保持工具集不变，将 `EnterPlanMode` / `ExitPlanMode` 设计成工具本身。

### 5. 大型工具集用 defer_loading

`defer_loading: true` 发送只含工具名的轻量 stub，模型通过 `ToolSearch` 工具按需加载完整 schema。始终保持相同 stub 顺序，前缀稳定。

### 6. Compaction 的 Cache-Safe Forking

Compaction call 使用与主对话完全相同的 system prompt、工具集，在历史消息末尾追加 compaction prompt。前缀几乎相同 → 缓存命中。

---

## 💡 关键洞察

- **前缀稳定性是第一设计约束**：系统架构从一开始就要围绕「保持 cache prefix 不变」来组织
- **Cache hit rate 是一级运营指标**：Claude Code 对命中率下降触发告警（SEV）

---

## 🔗 相关页面

- [[llm/concepts/prompt-caching/index]] — 跨来源综合的概念页（API 细节、实现代码）
- [[llm/entities/Thariq Shihipar]] — 作者
- [[llm/summaries/thariq-claude-code-writing-thread]] — 作者文章索引

## 📎 原文摘录

> "Cache Rules Everything Around Me — prompt caching is what makes long-running agentic products feasible."

> "We run alerts on our prompt cache hit rate and declare SEVs if they're too low."
