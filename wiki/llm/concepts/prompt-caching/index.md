---
title: "Prompt Caching"
type: concept
created: 2026-04-11
updated: 2026-04-11
sources:
  - https://platform.claude.com/docs/en/build-with-claude/prompt-caching
  - https://x.com/trq212
tags:
  - AI/LLM/Caching
---

# Prompt Caching

> Claude API 的 prompt caching 通过将请求前缀缓存在服务端，后续命中时按约 0.1x 价格计费，是长上下文 Agent 产品经济可行性的关键。

## 子页面

- [[llm/concepts/prompt-caching/api-breakpoint-management]] — API 断点管理、配额计算、多轮对话策略
- [[llm/concepts/prompt-caching/system-design-principles]] — 系统架构层：前缀稳定性、工具集管理、模型切换
- [[llm/concepts/prompt-caching/compaction-cache-safe-forking]] — Compaction 与 Cache-Safe Forking

## 核心机制

`cache_control` 是「前缀缓存断点」，标记位置及其之前的所有内容被整体哈希缓存。

- 每次请求最多 **4 个**显式 `cache_control`，覆盖 system + messages + tools 总和
- API 有 **20-block lookback**：当前断点 miss 时向前最多回溯 20 个 block 找旧缓存
- 4 个上限与服务端**自动缓存（Automatic Caching）完全无关**，两者独立

## 关键原则

1. **前缀稳定性是第一设计约束**
2. **Cache hit rate 是一级运营指标**
3. **任何破坏前缀的操作都有隐性成本**

## 来源

- [[llm/summaries/prompt-caching-is-everything]] — Thariq 的生产实践经验
- [[llm/summaries/claude-prompting-best-practices]] — 官方文档中的 caching 相关内容
