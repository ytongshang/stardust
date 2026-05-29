---
title: "The Importance of Agent Harness in 2026 (Philipp Schmid)"
type: summary
created: 2026-04-19
updated: 2026-04-19
sources:
  - "Philipp Schmid — https://www.philschmid.de/agent-harness-2026 (published 2026-01-05)"
tags: [agents, harness, bitter-lesson, 2026, benchmark, summary]
---

# The Importance of Agent Harness in 2026 (Philipp Schmid)

**来源**：Philipp Schmid，2026-01-05  
**原文**：[raw/llm/articles/agents-harness/philipp-schmid-agent-harness-2026.md](../../raw/llm/articles/agents-harness/philipp-schmid-agent-harness-2026.md)

---

## 核心命题

AI 开发已到达拐点：**管理长任务的基础设施比模型能力本身更重要**。

---

## Leaderboard 幻觉

静态 benchmark 掩盖了顶级模型之间的真实差距。

> "A 1% difference on a leaderboard cannot detect the reliability if a model drifts off-track after fifty steps."

真正的区分维度是**耐久性（durability）**：执行数百次工具调用后，模型能否不跑偏。标准 benchmark 完全无法测量这一点。

---

## Harness 的系统比喻

| 组件 | 类比 |
|------|------|
| Model | CPU（计算能力） |
| Context window | RAM（易失性内存） |
| Agent Harness | OS（治理层） |
| Agent | Application（用户逻辑） |

---

## Harness 的三大功能

1. **验证真实进展**：让用户用实际 use case 评估最新模型，而不是依赖 benchmark 分数
2. **赋能用户体验**：标准化 harness 让用户直接获得经过验证的最佳实践
3. **反馈驱动优化**：共享执行环境产生结构化数据，驱动 benchmark 的迭代改进

> "The ability to improve a system is proportional to how easily you can verify its output."

Harness 把模糊工作流转化为可测量、可评分的结构化数据。

---

## Bitter Lesson 压力

Rich Sutton 的"苦涩教训"：通用计算方法长期总是打败人类先验知识编码的方法。

在 agent harness 工程上的表现：

| 团队 | 案例 |
|------|------|
| **Manus** | 六个月内重构 harness **5 次** |
| **LangChain** | 一年内重构 Open Deep Research **3 次** |
| **Vercel** | 砍掉 **80%** 工具后，步骤更少、token 更少、响应更快 |

过度工程化的控制流随每次模型升级而过时。**轻量、灵活的 harness 才能在模型迭代中存活**。

---

## 未来方向

训练与推理的边界在收敛。Context 耐久性成为新瓶颈。Labs 会用 harness 精确检测模型在长任务中"漂移"的时间点，把这些数据反哺训练——开发对退化有抵抗力的模型。

---

## 开发者建议

1. **从简单开始**：构建健壮的原子工具，让模型自主规划。实现护栏、重试、验证，而不是复杂的控制流
2. **Build to Delete**：设计时预期代码会被替换。新模型会让现有逻辑过时
3. **Harness 即数据集**：竞争优势从 prompt 转向捕获的执行轨迹。每次指令跟随失败都是下一次训练的数据

---

## 关联概念

- [[llm/concepts/agents-harness/harness-engineering-lessons|Harness Engineering Lessons]] — Bitter Lesson 与轻量 harness 原则的详细展开
- [[llm/concepts/agents-harness/what-is-harness|What Is Harness]] — 耐久性作为核心需求的论述
- [[llm/memory/context-compression|Context Compression]] — Context durability 的技术实现
