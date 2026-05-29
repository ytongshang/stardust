---
title: Harness Engineering Lessons
type: concept
created: 2026-04-19
updated: 2026-04-19
sources:
  - "OpenAI — Harness Engineering: leveraging Codex in an agent-first world (openai.com)"
  - "Philipp Schmid — The importance of Agent Harness in 2026 (philschmid.de)"
  - "Martin Fowler — Harness Engineering for Coding Agent Users (martinfowler.com)"
tags: [agents, harness, engineering-lessons, best-practices]
---

# Harness Engineering Lessons

2025–2026 年，大厂在生产 coding agent 场景下积累的 harness 工程经验。核心规律：**harness 必须轻量、可演化，而不是一次性到位的复杂系统。**

---

## OpenAI — Codex 团队经验

来源：OpenAI Harness Engineering（openai.com），记录 Codex 团队构建 100 万行代码全 agent 生成项目的过程。

### AGENTS.md 的正确定位

**错误做法（早期）：** 把 AGENTS.md 写成百科全书——把所有关于代码库的知识都堆进去，期望 agent 看一遍就全懂。

**正确做法（迭代后）：** AGENTS.md 做**目录**（table of contents），真正的知识放在结构化的 `docs/` 目录，按模块组织。

原因：
- Agent 每次任务只需要部分知识，按需加载比一次性读全高效
- 结构化 `docs/` 可以被 agent 精确检索，减少噪音
- AGENTS.md 太长 → agent 注意力分散，关键规则被淹没

对应 [[llm/concepts/agents-harness/guides-vs-sensors|Guides vs Sensors]] 框架：AGENTS.md 是 Deterministic Guide，应该精简有效，不是越详细越好。

### "Golden Principles" 编码进仓库

**错误做法：** 每周花 20% 时间人工清理 agent 生成的低质量代码（"AI slop"）。

**正确做法：** 把代码质量原则（golden principles）编码进仓库——作为 linting rules、测试用例、代码模板、schema 约束。让 agent 在写代码时自动被这些约束引导，而不是事后人工清理。

本质：把人工审查转化为自动化 Sensors 和 Guides，降低边际维护成本。

### 轻量工具集

Vercel 工程团队的发现：**砍掉 80% 的工具之后，agent 完成同样任务的步骤更少、响应更快。**

原因：工具越多，agent 每次决策时要考虑的选项越多，推理负担越重，选错工具的概率也越高。

设计原则：**从最小工具集开始，只在有具体需求时添加新工具。**

---

## Manus & LangChain — Harness 是演化中的系统

来源：Philipp Schmid，The importance of Agent Harness in 2026（philschmid.de）。

### 重构是正常节奏

| 团队 | 重构频率 |
|------|---------|
| Manus | 六个月内重构 harness **五次** |
| LangChain | 一年内把 Open Deep Research 重构**三次** |

这不是失败，而是 harness 工程的正常节奏。

原因：
- 模型能力在快速演进，上个季度需要工具处理的事情，新模型可能直接就会了
- 任务复杂度提升，旧的 harness 设计成为瓶颈
- 工程团队对"什么是 agent 真正需要的"的理解在加深

**推论：** 不要在 harness 上过度投资一次性设计。优先保持 harness 的**可替换性**。

### Bitter Lesson 压力

Bitter Lesson（苦涩教训）来自 RL 领域：长期来看，依赖计算力规模化的方法总是打败依赖人类先验知识的方法。

对 harness 工程的含义：随着模型变强，过度工程化的 harness（大量硬编码规则、复杂的专用工具）反而成为负担。Bitter Lesson 压力要求 harness 保持轻量，让模型的通用能力发挥空间。

---

## Martin Fowler — 系统性框架

来源：Harness Engineering for Coding Agent Users（martinfowler.com，作者 Birgitta Böckeler）。

### 为 LLM 设计 Sensor 输出

最重要的洞察之一：**最有效的 sensors 是专门为 LLM 消费设计的工具**，不是直接复用面向人类的工具输出。

对比：

| | 面向人类的 linter 输出 | 面向 LLM 的 linter 输出 |
|---|---|---|
| 格式 | 详细错误描述 + 行号 | 简洁 + 明确的修复建议 |
| 上下文 | 假设人类有背景知识 | 包含足够的上下文让 LLM 独立修复 |
| 噪音 | 包含警告、提示等非关键信息 | 过滤到只剩必须处理的错误 |

实践建议：在 CI/CD 之外，专门为 agent 维护一套 sensor 脚本，输出格式专为 LLM 优化。

### 不要把所有控制都推给推理型

计算型控制（linter、类型检查）的信噪比高、成本低，应该作为**主力护栏**，而不是觉得"LLM 够聪明，不需要这些"。

推理型控制（LLM-as-judge）适合：
- 计算型工具无法覆盖的语义判断
- 关键检查点的最终把关
- 不适合高频场景

---

## 综合原则

综合以上经验，distilled 为五条：

1. **轻量优先**：最小工具集 + 精简 guides，避免"越多越好"的直觉
2. **Guides 早于 Sensors**：在上游设置约束比在下游纠错便宜
3. **为 LLM 设计反馈格式**：sensor 输出专门针对 LLM 优化，不是复用人类工具
4. **harness 会反复重构**：优先可替换性，不要过度设计
5. **编码原则，而非依赖人工**：把质量约束变成自动化工具，不靠人力维护

---

## 相关页面

- [[llm/concepts/agents-harness/guides-vs-sensors|Guides vs Sensors]] — 本页经验的理论框架
- [[llm/concepts/agents-harness/core-components|Core Components]] — 经验对应到具体组件
- [[agent-tool-design]] — 轻量工具集的设计原则
