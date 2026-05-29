# Index — LLM / AI / Agent

> AI / LLM / Agent 系统领域的个人知识库。由 LLM 编译，人工审核。

## 🔖 Navigation
- [[#Concepts]] · [[#Memory]] · [[#Entities]] · [[#Summaries]] · [[#Open Questions]]

---

## Concepts

### Agent / Architecture
- [[llm/concepts/agent-file-system]] — 文件系统作为 Agent 状态抽象层（Unix 哲学 × VFS 工程实现）
- [[llm/concepts/agent-tool-design]] — Agent 工具设计原则：匹配模型能力、Progressive Disclosure
- [[llm/concepts/agent-skills]] — Skills 机制：模块化能力扩展、三层加载架构
- [[llm/concepts/multi-agent-workflow/index|Multi-Agent Workflow]] — (folder-split) 个人生活/工作自动化的 S0/S1/S2 多 Agent 工作流：最小 Agent 集合、checkpoint、迁移决策
- [[llm/concepts/agents-harness/index|Agents Harness]] — (folder-split) Agent = Model + Harness：组件拆解、Guides/Sensors 框架、大厂工程经验
    - [[llm/concepts/agents-harness/what-is-harness|What Is Harness]] — 基本概念、耐久性、与传统工程的区别
    - [[llm/concepts/agents-harness/core-components|Core Components]] — 七大核心组件（文件系统、Git、Bash、Agent Loop、MCP、记忆、子 Agent、权限）
    - [[llm/concepts/agents-harness/guides-vs-sensors|Guides vs Sensors]] — 前馈引导 × 反馈感知 × 计算型 × 推理型 四象限框架
    - [[llm/concepts/agents-harness/harness-engineering-lessons|Harness Engineering Lessons]] — OpenAI/Manus/Vercel/LangChain 工程经验与教训

### LLM / Caching
- [[llm/concepts/prompt-caching/index|Prompt Caching]] — (folder-split) Claude API prompt caching 原理、设计约束与实现
    - [[llm/concepts/prompt-caching/system-design-principles]] — 系统架构层：前缀稳定性、工具集管理、模型切换
    - [[llm/concepts/prompt-caching/api-breakpoint-management]] — API 断点管理、配额计算、多轮对话策略
    - [[llm/concepts/prompt-caching/compaction-cache-safe-forking]] — Compaction 与 Cache-Safe Forking

### LLM / Prompt
- [[llm/concepts/claude-prompting]] — Claude 4.x Prompting 综合：通用原则、Adaptive Thinking、Agentic 系统

---

## Memory

> (folder-split) Context window 管理：压缩、记忆持久化、召回检索。源自 Claude Code / OpenClaw / OpenHarness 三系统工程实现。

- [[llm/memory/index|Context Window Management]] — 总览与通用架构
- [[llm/memory/context-compression|Context Compression]] — 渐进式四级压缩流水线（微压缩 → 折叠 → Session Memory → LLM Compact）
- [[llm/memory/memory-persistence|Memory Persistence]] — 会话内记忆提取与跨会话持久化（文件系统 / SQLite / 快照）
- [[llm/memory/recall-retrieval|Recall & Retrieval]] — 记忆召回策略对比（LLM 语义选择 / 向量混合 / 关键词打分）
- [[llm/memory/implementation-comparison|Implementation Comparison]] — 三系统设计哲学横向对比

---

## Entities

- [[llm/entities/Thariq Shihipar]] — Anthropic Claude Code 团队成员，Agent / Caching / Skills 实践者

---

## Summaries (by topic)

### Agent / Architecture
- [[llm/summaries/files-are-all-you-need]] — Unix 哲学与 Agentic AI 系统设计（arXiv paper）
- [[llm/summaries/seeing-like-an-agent]] — 构建 Claude Code 的工具设计经验（Anthropic blog）
- [[llm/summaries/thariq-claude-code-writing-thread]] — Thariq 技术写作合集索引（blog）

### Agent / Harness
- [[llm/summaries/agents-harness/langchain-anatomy-agent-harness]] — Agent = Model + Harness 拆解：文件系统、bash、sandbox、上下文管理（LangChain）
- [[llm/summaries/agents-harness/martin-fowler-harness-engineering]] — Guides vs Sensors × 计算型 vs 推理型四象限框架（Thoughtworks / martinfowler.com）
- [[llm/summaries/agents-harness/philipp-schmid-agent-harness-2026]] — Bitter Lesson、耐久性测量、轻量 harness 原则（Philipp Schmid，2026-01-05）
- [[llm/summaries/agents-harness/parallel-what-is-agent-harness]] — Context lifecycle、Orchestrator/Harness 分工、model-agnostic 特性（Parallel.ai）

### Agent / Skills
- [[llm/summaries/how-we-use-skills]] — Claude Code 团队 Skills 实战经验（Anthropic blog）
- [[llm/summaries/claude-agent-skills-docs]] — Anthropic 官方 Agent Skills 文档笔记

### LLM / Caching
- [[llm/summaries/prompt-caching-is-everything]] — Prompt Caching 生产实践（Thariq blog）

### LLM / Prompt
- [[llm/summaries/claude-prompting-best-practices]] — Claude 4.x Prompting 最佳实践（Anthropic docs）

### Systems
- [[llm/summaries/nodejs-vfs-pr]] — Node.js 内存文件系统 PR 分析

---

## Open Questions

- Agent 系统在大规模工具集下如何保持 cache prefix 稳定？
- llm-wiki 模式在团队场景下的 audit 工作流如何设计？
- 待精读：raw/articles/trq212-agent-file-system — Your Agent should use a File System
- 待精读：raw/articles/trq212-agents-need-bash — Why even non-coding agents need bash
