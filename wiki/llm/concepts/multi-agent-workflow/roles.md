---
title: Multi-Agent Workflow — Roles
type: concept
created: 2026-05-27
updated: 2026-05-27
sources:
  - "Slock thread #life:a361fe23 — 多 Agent 工作流设计讨论 (2026-05-27)"
tags:
  - agent
  - workflow
  - roles
  - coordinator
---

# Multi-Agent Workflow — Roles

> 上级页面：[[llm/concepts/multi-agent-workflow/index|Multi-Agent Workflow]]

## 三个真实 Agent

### Coordinator / Planner

职责：

- 接收用户自然语言需求。
- 判断任务等级 S0 / S1 / S2。
- 生成任务卡：目标、边界、不做什么、验收标准、风险。
- 控制 checkpoint 和用户沟通。
- 汇总 Builder / Reviewer 结果。
- 最后执行迁移决策：thread、repo docs、Obsidian wiki、MEMORY、skill 分别写什么。

约束：

- S0 可以同时担任 Builder。
- S1 / S2 不直接 commit 代码，避免失去全局视角。
- 澄清问题最多 1-3 个。
- 不预测 Builder/Reviewer 结果，等待真实回报。

### Builder

职责：

- 看代码、确认现有模式。
- 实现改动，保持小步提交。
- 写实现总结。
- 根据 Reviewer 反馈修复问题。

约束：

- 改代码前必须检查 git 状态。
- 不回滚用户已有改动。
- 小步提交，便于回退。
- 如果卡住，先回 Coordinator/Architect 重新确认方案；只有发现需求理解错了才回到用户澄清。

### Reviewer / Verifier

职责：

- 站在找问题的角度做 review。
- 跑 typecheck、测试、dry-run、必要时端到端验证。
- 输出验证报告和风险。

约束：

- 不和 Builder 同时改同一批文件。
- 可提出 patch，但默认不直接 merge。
- 对失败测试必须给出具体复现命令和错误摘要。

## 职责压缩

| 原职责 | 压缩到真实 Agent |
|---|---|
| Planner | Coordinator |
| Architect | Coordinator（必要时可用 Plan subagent 辅助） |
| Scout | Builder（必要时可用 Explore subagent 辅助） |
| Builder | Builder |
| Reviewer | Reviewer |
| Librarian | Coordinator |

Planner、Architect、Scout、Librarian 是工作阶段，不一定是常驻 Slock 身份。

## 与 Claude Code subagent 的关系

Slock Agent 是沟通身份；Claude Code subagent 是临时执行体。

- S1/S2 中，Coordinator 可以启动临时 Builder subagent。
- Builder 可用 Explore/Plan 辅助，但最终对代码改动负责。
- Reviewer 可用 code-review skill 辅助，但最终对验证报告负责。

