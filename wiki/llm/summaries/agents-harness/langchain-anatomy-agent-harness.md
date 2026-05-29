---
title: "The Anatomy of an Agent Harness (LangChain)"
type: summary
created: 2026-04-19
updated: 2026-04-19
sources:
  - "Vivek Trivedy, LangChain Blog — https://www.langchain.com/blog/the-anatomy-of-an-agent-harness"
tags: [agents, harness, langchain, agent-architecture, summary]
---

# The Anatomy of an Agent Harness (LangChain)

**来源**：Vivek Trivedy，LangChain Blog  
**原文**：[raw/llm/articles/agents-harness/langchain-anatomy-agent-harness.md](../../raw/llm/articles/agents-harness/langchain-anatomy-agent-harness.md)

---

## 核心命题

$$\text{Agent} = \text{Model} + \text{Harness}$$

Harness 是"模型之外的所有代码、配置和执行逻辑"。这篇文章是从第一性原理推导 harness 各组件存在理由的最清晰入门文章，也是 [[llm/concepts/agents-harness/index|Agents Harness]] 主题的奠基来源之一。

---

## 为什么模型离不开 Harness

模型天然不具备：
- 跨会话的持久状态
- 自主执行代码的能力
- 访问实时信息的途径
- 配置环境、安装依赖的能力

Harness 填补这些空白。

---

## 核心组件拆解

### 文件系统 + Git

文件系统是"天然的多 agent 协作界面"（natural collaboration surface）：
- 持久化跨会话的工作状态
- Git 加版本控制：rollback 错误、分支实验、多 agent 通过 worktree 并行互不干扰

### Bash / 通用代码执行

不要为每个可能的场景预定义工具——给 agent 一个 bash 工具让它自主解决。计算自主性（computational autonomy）优先于穷举所有工具。

### Sandboxes

隔离执行环境：安全运行 agent 生成的代码、可扩展并行任务、预装语言运行时。

### 上下文管理三策略

| 策略 | 机制 |
|------|------|
| **Compaction** | 智能摘要旧上下文，防止 context window 溢出 |
| **Tool call offloading** | 大体积工具输出存文件，上下文只保留关键摘录 |
| **Skills (Progressive Disclosure)** | 初始化时不暴露全部工具 schema，按需动态加载 |

### Ralph Loop 模式

长任务跑偏时的纠正机制：在新的干净上下文中重新注入原始 prompt，强制 agent 对齐原始目标。

---

## 前沿研究方向（LangChain deepagents）

- 数百个并行 agent 协作同一代码库
- Agent 分析自己的 trace 来定位 harness 层面的故障
- 动态即时工具 + 上下文组装（just-in-time assembly）

---

## 重要洞察：Harness 与模型训练的共演化

模型可能对特定 harness 实现过拟合——"改变工具逻辑会导致模型性能下降"。最优 harness 不一定是模型训练时用的那个，不同 harness 配置在相同模型上产生截然不同的 benchmark 结果。

---

## 关联概念

- [[llm/concepts/agents-harness/core-components|Core Components]] — 本文各组件的扩展讨论
- [[llm/concepts/agents-harness/what-is-harness|What Is Harness]] — 定义层讨论
- [[agent-tool-design]] — Progressive Disclosure 工具设计原则
- [[llm/memory/context-compression|Context Compression]] — Compaction 机制详解
