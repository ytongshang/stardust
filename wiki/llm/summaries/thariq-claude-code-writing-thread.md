---
title: "Thariq (@trq212) 技术写作合集 —— Claude Code & Agent 构建实践"
type: summary
created: 2026-03-25
updated: 2026-04-11
sources:
  - https://x.com/trq212/status/2035372716820218141
tags:
  - AI/Agent/Architecture
  - AI/Agent/Skills
  - AI/LLM/Caching
origin_type: blog
author: "Thariq Shihipar"
rating: ⭐⭐⭐⭐⭐
---

# Thariq (@trq212) 技术写作合集 —— Claude Code & Agent 构建实践

## 📌 核心观点

> Thariq 是 Anthropic Claude Code 团队成员，这是他所有技术文章的置顶索引帖。内容覆盖 Agent 构建的核心设计决策：工具空间、缓存、文件系统、Bash、SDK、Skills。每篇都是来自生产实践的第一手经验。

---

## 📋 文章索引

| 文章 | 核心论点 | 对应笔记 |
|------|---------|---------|
| How We Use Skills | "skills are the abstraction that all agents will build on" | [[llm/summaries/how-we-use-skills]] |
| Prompt Caching Is Everything | "Cache Rules Everything Around Me" | [[llm/summaries/prompt-caching-is-everything]] |
| Seeing like an Agent | "building an agent is more of an art than a science" | [[llm/summaries/seeing-like-an-agent]] |
| Your Agent should use a File System | "This is a hill I will die on." | raw/articles/trq212-agent-file-system（待精读） |
| Why even non-coding agents need Bash | "bash is all you need" | raw/articles/trq212-agents-need-bash（待精读） |
| Building agents with the Claude Agent SDK | SDK 入门最佳路径 | — |
| Making Playgrounds using Claude Code | 可视化迭代工具 | — |
| Spec 生成技巧（AskUserQuestionTool） | 访谈式需求澄清 | — |

---

## 💡 跨文章核心洞察

- **Skills 是 Agent 能力封装的基本单元**，未来所有 Agent 框架可能都会走向类似机制
- **Prompt Caching 是 Agent 经济可行性的关键**，系统架构从设计阶段就要围绕缓存命中率建模
- **文件系统是 Agent 的天然状态层**：可持久化、可检查、可 debug，无需额外数据库
- **Bash 优先**：给 Agent 加 Bash 工具比专门封装的 API 工具更灵活、更不容易幻觉

---

## 🔗 相关页面

- [[llm/entities/Thariq Shihipar]] — 作者简介
- [[llm/summaries/how-we-use-skills]]
- [[llm/summaries/prompt-caching-is-everything]]
- [[llm/summaries/seeing-like-an-agent]]

## 📎 原文摘录

> "skills are the abstraction that all agents will build on"
> "imo my highest alpha post is on prompt caching, but it's only really relevant if you're building agents from scratch"
> "This is a hill I will die on. Every agent can use a file system."
> "bash is all you need"
