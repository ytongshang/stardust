---
title: Research Collection Workflow
type: process
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent
tags:
  - agent-harness
  - research
---

# Research Collection Workflow

## 目标

收集不是为了每天攒链接，而是为了理解 agent harness 的概念、实现边界和新的设计想法。

来源可以是：

- GitHub repo / PR / issue。
- X/Twitter thread。
- blog / engineering article。
- paper / benchmark。
- release notes。
- demo / talk。

## 收集时先归类

先判断材料属于哪个概念：

- Agent Loop
- Event Log
- Handoff & Transfer
- Input Transform
- Prompt Assembly
- Context Injection
- Context Budgeting
- Compaction
- Tool Definition
- Tool Execution
- Tool Results
- Tool Error Recovery
- Tool Interruption
- Tool Availability
- Tool Permissions
- Remote Tools
- Extension Registry
- Skills
- Workspace
- Memory
- Subagents
- Background Agents
- Agent Communication
- Lifecycle
- Guardrails
- Observability / Evals

如果找不到分类，先写到 `harness/index.md` 的 Open Questions，不要马上新建大类。

## 去重规则

如果外部材料只是覆盖已有概念：

- 不单独写长文。
- 在对应概念目录下简短记录，或追加到该目录已有外部实现笔记。
- 简短说明它的命名和 uxarts-agent 对应关系。

如果外部材料有新东西：

- 写进对应概念目录。
- 新增 `External Ideas` 或 `Implementation Notes` 小节。
- 说明它解决了什么问题，和 uxarts-agent 的差异是什么。
- 外部实现笔记放到 `implementations/<source>.md`，不要和概念 `index.md` 混在一起。

## 判断“值得写”的标准

优先记录：

- 新的抽象边界。
- 更好的上下文注入/压缩策略。
- 更强的 tool policy / permission。
- 更清晰的 workspace backend。
- 更可靠的 pending/resume/lifecycle。
- 更好的 sub-agent lifecycle 或 merge。
- 可复用 trace/eval/replay 方法。
- 生产事故恢复经验。

可以跳过：

- 只是又一个 chat-with-tools demo。
- 只是把文件系统、memory、subagent 重新命名。
- 没有实现细节的宣传页。
- 只展示效果、不解释 harness 设计的项目。

## 建议记录格式

```markdown
## Source

- URL:
- Type: GitHub / X / article / paper / release
- Date checked:

## Belongs To

- Concept:
- Existing uxarts-agent equivalent:

## What Is New

- 

## Implementation Detail

- 

## Keep / Skip

- Keep because:
- Skip because:
```

## 与自动化的关系

不需要每天为了收集而收集。更合适的是：

- 每周或需要时扫一轮。
- 看到强信号再深入。
- 每次只围绕一个概念补材料。
- 自动化只负责候选发现和去重，不负责强行产出。

已有记录：

- `automations/agent-harness-radar/checked-links.md`

后续可以把 automation prompt 改成“按概念发现候选”，而不是“每日项目雷达”。
