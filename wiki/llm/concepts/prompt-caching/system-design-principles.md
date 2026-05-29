---
title: "Prompt Caching — System Design Principles"
type: concept
created: 2026-04-11
updated: 2026-04-11
sources:
  - https://x.com/trq212
  - https://platform.claude.com/docs/en/build-with-claude/prompt-caching
tags:
  - AI/LLM/Caching
---

# Prompt Caching — System Design Principles

系统架构层面的缓存设计原则，来自 Claude Code 的生产经验。

## Prompt 布局顺序

静态内容在前，动态内容在后：

```
① Base System Prompt & Tools    ← globally cached（跨所有 session）
② CLAUDE.md / 项目上下文        ← cached per project
③ Session Context               ← cached per session
④ Conversation Messages         ← 每轮动态增长
```

**常见破坏顺序的失误：**
- 在静态 system prompt 里插入时间戳或当前日期
- 工具定义非确定性排序（map 遍历顺序不稳定等）
- 中途更新 AgentTool 可调用的子 agent 列表

**动态信息的正确传递方式：** 在下一轮 user message 或 tool result 中插入 `<system-reminder>` 标签。

```python
# ❌ 破坏缓存
system_prompt = f"Current date: {today}. You are a helpful assistant..."

# ✅ 保持缓存
# system_prompt 保持静态
# 在下一轮 user message 末尾追加：
# <system-reminder>Today's date is {today}.</system-reminder>
```

## 工具集管理

工具定义属于缓存前缀，中途增删必破坏缓存。

**Plan Mode 的正确实现：**
```
❌ 进入 plan mode 时替换工具集为只读工具
✅ 始终保持完整工具集，将 EnterPlanMode / ExitPlanMode 设计为工具本身
```

**大型工具集的 defer_loading 模式：**
```
❌ 按需删除不需要的工具（会破坏缓存）
✅ 使用 defer_loading: true 发送轻量 stub（只有工具名）
   模型通过 ToolSearch 工具按需加载完整 schema
```

## 模型切换

Prompt cache 与模型绑定，会话中途切换模型会使缓存失效并需要重建。

```
❌ 会话中途从 Opus → Haiku（100k token 的缓存全部失效）
✅ 用 subagent 完成切换：主模型准备「交接消息」，由子 agent 用另一个模型承接
```

## 监控

Cache hit rate 是一级运营指标，应当像监控服务可用性一样监控它，命中率下降即告警。

## 反模式

> ❌ 在静态 system prompt 里写入动态信息
> ❌ 进入 plan mode 时替换工具集
> ❌ 会话中途切换模型
> ❌ compaction 时使用独立 API 调用（不同 system prompt）

## 相关页面

- [[llm/concepts/prompt-caching/index]] — 概览
- [[llm/concepts/prompt-caching/api-breakpoint-management]] — API 层面的断点管理
- [[llm/concepts/prompt-caching/compaction-cache-safe-forking]] — Compaction 场景
