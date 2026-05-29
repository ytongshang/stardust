---
title: "Harness Engineering for Coding Agent Users (Martin Fowler)"
type: summary
created: 2026-04-19
updated: 2026-04-19
sources:
  - "Birgitta Böckeler (Thoughtworks), martinfowler.com — https://martinfowler.com/articles/harness-engineering.html"
tags: [agents, harness, martin-fowler, guides-sensors, thoughtworks, summary]
---

# Harness Engineering for Coding Agent Users (Martin Fowler)

**来源**：Birgitta Böckeler，Thoughtworks，发布于 martinfowler.com  
**原文**：[raw/llm/articles/agents-harness/martin-fowler-harness-engineering.md](../../raw/llm/articles/agents-harness/martin-fowler-harness-engineering.md)

---

## 核心贡献

这篇文章是目前对 harness 控制层最系统的分类框架，引入了 **Guides vs Sensors × 计算型 vs 推理型** 四象限模型。完整框架见 [[llm/concepts/agents-harness/guides-vs-sensors|Guides vs Sensors]]。

---

## Guides（引导）vs Sensors（传感器）

| 维度 | Guides | Sensors |
|------|--------|---------|
| 方向 | 前馈 — 行动前引导 | 反馈 — 行动后观察 |
| 目标 | 第一次就做对 | 发现偏差后自我纠正 |
| 例子 | 架构文档、编码规范 | linter、测试、代码审查 |

**关键原则**：只有 sensors 会重复犯错；只有 guides 永远不知道效果如何。两者缺一不可。

---

## 计算型 vs 推理型

| 类型 | 特点 | 适用 |
|------|------|------|
| **Computational** | 确定性、毫秒级、CPU | Linter、类型检查、结构测试 |
| **Inferential** | 语义分析、昂贵、GPU | AI 代码审查、LLM-as-judge |

---

## 三类 Harness 维度

### 1. 可维护性 Harness（Maintainability）
检测代码内部质量：重复代码、圈复杂度、测试覆盖率。
**盲区**：无法识别需求误解、功能蔓延、过度设计（需要推理型 sensor）。

### 2. 架构适配度 Harness（Architecture Fitness）
基于 fitness function 原则——把架构约束转化为可测量的指标并持续监控。

### 3. 行为正确性 Harness（Behaviour）
**目前最弱的领域**。Feedforward：功能规格说明；Feedback：AI 生成测试套件。
核心问题：agent 自己写测试 + 自己执行 → 自我验证循环，不可靠。

---

## 时机策略：Keep Quality Left

| 阶段 | 控制类型 | 典型工具 |
|------|----------|---------|
| 提交前 | 快速计算型 | linter、基础测试、初步代码审查 |
| CI/CD | 昂贵计算型 + 推理型 | mutation testing、全面代码审查 |
| 持续监控 | 漂移检测 | dead code、覆盖率质量、SLO 降级 |

---

## 重要新概念

**Harnessability**：代码库对控制实现的支持程度。强类型语言、清晰模块边界、有主张的框架（opinionated frameworks）提升可 harness 性。

**Ambient Affordances**（环境可供性）：agent 环境的结构性特质，使其"可读、可导航、可处理"。技术选型、架构模式的选择本身就是一种隐式 guide。

**Harness Templates**：把通用的架构拓扑（API 服务、事件处理、数据仪表盘）包装成预配置的 guides + sensors 包，加速团队采用。

---

## 实际案例

- **OpenAI**：分层架构 + 自定义 linter + 结构测试 + 定期"垃圾回收"扫描（检测漂移）
- **Stripe Minions**：pre-push hooks 运行启发式选择的 linter，强调 shift left
- **Thoughtworks 团队**：混合计算型/推理型 sensor 应对架构漂移；"janitor army" 代码质量倡议

---

## 尚未解决的挑战

1. **行为验证**：如何对功能正确性建立足够信心？
2. **Harness 一致性**：guides 和 sensors 增多后如何保持同步？
3. **矛盾指令**：agent 如何处理相互冲突的 guides 和 sensors？
4. **Harness 覆盖率**：如何评估 harness 本身的充分性？（类比 mutation testing 评估测试套件）

---

## 关联概念

- [[llm/concepts/agents-harness/guides-vs-sensors|Guides vs Sensors]] — 本文框架的完整展开
- [[llm/concepts/agents-harness/harness-engineering-lessons|Harness Engineering Lessons]] — 本文实际案例汇总进大厂教训
- [[agent-tool-design]] — Ambient Affordances 与工具设计的关系
