---
title: "Seeing Like an Agent: Lessons from Building Claude Code"
type: summary
created: 2026-03-26
updated: 2026-04-11
sources:
  - https://www.anthropic.com/engineering/claude-code-tool-use
tags:
  - AI/Agent/Architecture
origin_type: blog
author: "Anthropic (Claude Code Team)"
---

# Seeing Like an Agent: Lessons from Building Claude Code

## 📌 核心观点

> 构建 Agent 的工具设计是一门艺术，不是科学。工具要契合模型的能力边界，随着模型变强要及时调整工具，而不是一成不变。核心方法论：**像 Agent 一样思考**——观察输出、持续实验、理解模型的"视角"。

---

## 📋 三个真实案例

### 案例 1：AskUserQuestion 工具的进化

- **尝试一（失败）**：给 ExitPlanTool 加 questions 数组参数 → Claude 同时输出 plan 和问题，逻辑混乱
- **尝试二（不稳定）**：让 Claude 输出特定 markdown 结构 → Claude 随意附加内容，不可靠
- **尝试三（成功）**：创建专用 `AskUserQuestion` 工具，触发时展示 modal、阻塞等待回答 → 结构化输出有保障

**洞察**：工具要让模型觉得"自然"，强迫的结构约束不如设计成模型愿意主动调用的工具。

### 案例 2：TodoWrite → Task Tool（随模型能力升级工具）

早期用 `TodoWrite` + 每 5 轮系统提醒，随着模型变强，反而成了约束——Claude 认为必须严格执行列表，缺乏灵活性。

演进为 `Task Tool`：支持依赖关系、多 Agent 共享更新、模型可主动修改任务。

**洞察**：模型能力提升后，原来帮助模型的工具可能反过来限制它。要**持续重新审视工具假设**。

### 案例 3：Progressive Disclosure（搜索与上下文构建）

演进路径：RAG 向量数据库 → Grep 工具（主动搜索）→ Agent Skills（递归文件引用）

Skills 实现**渐进式披露**：Claude 读 skill 文件，文件引用更多文件，递归探索。无需增加新工具就能扩展能力。

---

## 结论

- 工具设计要**匹配模型能力**，像设计数学解题工具一样（纸 → 计算器 → 电脑）
- **持续观察模型输出**，模型能力提升后工具要同步迭代
- **Progressive Disclosure** 是扩展 Agent 能力的强力手段，避免工具数量膨胀

---

## 🔗 相关页面

- [[llm/summaries/thariq-claude-code-writing-thread]] — Thariq 合集索引，含本文背景
- [[llm/summaries/how-we-use-skills]] — Skills 设计的延伸
- [[llm/summaries/prompt-caching-is-everything]] — 同系列，Caching 视角
- [[llm/concepts/agent-tool-design]] — （待建）Agent 工具设计原则综合

## 📎 原文摘录

> "Designing the tools for your models is as much an art as it is a science."

> "As model capabilities increase, the tools that your models once needed might now be constraining them."
