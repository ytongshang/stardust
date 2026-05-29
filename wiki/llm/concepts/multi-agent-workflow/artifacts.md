---
title: Multi-Agent Workflow — Artifacts
type: concept
created: 2026-05-27
updated: 2026-05-27
sources:
  - "Slock thread #life:a361fe23 — 多 Agent 工作流设计讨论 (2026-05-27)"
tags:
  - agent
  - workflow
  - artifacts
  - skills
  - memory
---

# Multi-Agent Workflow — Artifacts

> 上级页面：[[llm/concepts/multi-agent-workflow/index|Multi-Agent Workflow]]

## 标准产物

### Work Order

```md
## Work Order

User intent:
Goal:
Non-goals:
Inputs:
Outputs:
Acceptance criteria:
Risk:
Task level: S0 | S1 | S2
Agent plan:
```

### Tech Plan

```md
## Tech Plan

Recommended path:
Alternatives:
1.
2.
3.
Reuse existing:
New components:
Testing plan:
Open questions:
```

### Code Map

```md
## Code Map

Repo:
Current git status:
Relevant files:
Existing patterns:
Files to edit:
Files to avoid:
Verification commands:
```

### Verification Report

```md
## Verification

Commands run:
Results:
E2E / dry-run:
Known risks:
Ready for review: yes | no
```

### Migration Decision

```md
## Migration Decision

Promote to repo docs:
Promote to Obsidian wiki:
Promote to Agent MEMORY:
Promote to skill:
Discard as task-local:

Pattern status:
- [ ] one-off
- [ ] pattern-candidate
- [ ] reusable skill
```

## 迁移规则

| 内容类型 | 写到哪里 | 判断标准 |
|---|---|---|
| 本次讨论、临时草稿、一次性决策 | Slock thread | 只对本次任务有用 |
| 项目命令、代码模式、避坑指南 | repo docs / README / notes | 以后维护同一项目会复用 |
| 用户偏好、工作方式、长期上下文 | Agent `MEMORY.md` / notes | 影响以后如何服务用户 |
| 模型需要主动调用的可执行能力 | skill | 以后会重复触发，并且需要固定命令/流程 |
| 研究性概念、架构方法、通用流程 | Obsidian wiki | 给人和模型长期阅读、复盘、演化 |

YAGNI 规则：

- 第一次出现的模式：只记为 `[pattern-candidate]`。
- 第二次出现，或已经明确会有多种来源/多次复用：抽成 skill 或模板。
- 没有明确迁移动作的 thread 内容，默认用完即抛。

## Skill 化建议

本 workflow 本身适合沉淀为 `multi-agent-workflow` skill。Skill 应该短，给模型执行用；wiki 页面保留完整解释。

Skill 最小内容：

- S0/S1/S2 路由规则。
- Coordinator / Builder / Reviewer 职责。
- Checkpoint 压缩规则。
- 回退协议。
- Migration Decision 模板。
- 明确提醒：优先选择最小足够流程，不要为了使用 skill 而创建额外 Agent。

