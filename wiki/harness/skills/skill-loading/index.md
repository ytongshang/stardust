---
title: Skill Loading
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - skills
---

# Skill Loading

Skill Loading 是把 skill 的完整说明、示例、脚本或资源按需放进 agent 可读上下文的过程。

## 它解决什么问题

Skill Discovery 只能告诉 agent 有某个 skill；真正使用时还需要完整 instructions、examples、scripts、templates、assets 或 reference docs。Skill Loading 负责把这些材料按需放到 agent 可读的位置，同时控制 context cost。

它需要处理：

- 加载哪些文件：`SKILL.md`、references、scripts、templates、examples。
- 何时加载：run start、模型请求、tool call 前、sub-agent start。
- 加载到哪里：prompt context、workspace files、tool-accessible resources。
- 多个 skill 冲突时如何排序和合并。
- 加载记录如何进入 trace，方便解释 agent 为什么遵循某个 skill。

## 常见做法

常见实现会使用 progressive disclosure：

- 先在 prompt 中放 skill summary 和触发规则。
- 模型选择 skill 后读取完整 `SKILL.md`。
- 只有需要时再读取 references 或 materialize assets。
- 对大文件做 budget control，只加载和当前任务相关的部分。
- 把 loaded skill 列入 run state，避免重复加载或版本漂移。

## 边界

- [[harness/skills/skill-discovery/index|Skill Discovery]] 决定是否应该看 skill；Skill Loading 负责把内容真正带入运行。
- [[harness/skills/skill-materialization/index|Skill Materialization]] 负责把 skill package 落成本地/远端资源；Loading 负责读取和注入。
- [[harness/context/context-budgeting/index|Context Budgeting]] 决定 skill 内容能占多少上下文。

## 目录

- [[harness/skills/skill-loading/implementations/uxarts-agent|uxarts-agent 实现]]
