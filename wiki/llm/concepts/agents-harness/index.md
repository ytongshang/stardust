---
title: Agents Harness
type: concept
created: 2026-04-19
updated: 2026-04-19
sources:
  - "LangChain — The Anatomy of an Agent Harness (blog.langchain.com)"
  - "Martin Fowler — Harness Engineering for Coding Agent Users (martinfowler.com)"
  - "OpenAI — Harness Engineering: leveraging Codex in an agent-first world (openai.com)"
  - "Philipp Schmid — The importance of Agent Harness in 2026 (philschmid.de)"
  - "Parallel — What is an Agent Harness (parallel.ai)"
tags: [agents, harness, agent-architecture, infrastructure]
---

# Agents Harness

> **公式**：Agent = Model + Harness

Harness 是模型之外的所有代码、配置和执行逻辑，让模型的智能在真实任务中变得可用。模型提供推理能力，harness 提供耐久性（durability）——两者缺一，agent 无法完成长任务。

2025–2026 年，随着 coding agent 进入生产环境，harness 工程（harness engineering）成为 AI agent 基础设施的核心命题。顶级模型在静态 benchmark 上差距在缩小，但真正区分 agent 能力的是 harness 的质量。

---

## 子页面

| 页面 | 内容 |
|------|------|
| [[llm/concepts/agents-harness/what-is-harness\|What Is Harness]] | 基本概念、定义、与传统软件工程的区别 |
| [[llm/concepts/agents-harness/core-components\|Core Components]] | Harness 的七大核心组件拆解 |
| [[llm/concepts/agents-harness/guides-vs-sensors\|Guides vs Sensors]] | Martin Fowler 框架：前馈引导与反馈感知的四象限模型 |
| [[llm/concepts/agents-harness/harness-engineering-lessons\|Harness Engineering Lessons]] | 大厂工程经验与教训（OpenAI、Manus、Vercel、LangChain） |

---

## 关键结论（摘要）

- **轻量优先**：Vercel 砍掉 80% 工具后，步骤更少、响应更快。Bitter Lesson 压力下，harness 必须保持精简。
- **AGENTS.md ≠ 百科全书**：AGENTS.md 做"目录"，真正的知识放在结构化 `docs/` 目录。
- **Guides + Sensors 双轨**：用计算型工具（linter、类型检查）做快速护栏，用推理型工具（LLM-as-judge）做高价值判断。
- **Harness 会频繁重构**：Manus 六个月重构五次，LangChain 一年重构三次——这是正常节奏，不是失败。

---

## 关联概念

- [[agent-skills]] — Skills 是 harness 按需加载能力的核心机制
- [[agent-tool-design]] — 工具设计影响 harness 的工具注册层
- [[agent-file-system]] — 文件系统是 harness 最基础的状态存储原语
- [[llm/memory/index|Context Window Management]] — 记忆与上下文压缩是 harness 的长期记忆层
