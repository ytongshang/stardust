---
title: Skill Discovery
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - skills
---

# Skill Discovery

Skill Discovery 是让模型知道有哪些 skill 可用、什么时候应该查看或启用 skill 的机制。它通常通过 catalog、摘要、触发规则或能力索引实现。

## 它解决什么问题

Skill 太多时不能全部塞进 prompt。Skill Discovery 让 agent 在低成本上下文里知道“有哪些能力可能有用”，再按任务需要决定是否加载完整 skill。

它需要回答：

- skill catalog 如何表示：名称、描述、触发条件、输入限制、资源位置。
- 模型什么时候应该发现 skill，什么时候应该直接执行任务。
- discovery 摘要如何足够短，同时不丢关键能力边界。
- skill 是否按 workspace、project、user、plugin 状态过滤。
- discovery 决策如何记录，方便 debug 为什么某个 skill 没被使用。

## 常见做法

常见实现包括：

- catalog summary：把所有 skill 的短描述放入 system/developer context。
- trigger rules：用关键词、file type、task type 或 model routing 触发。
- search index：通过 embedding/BM25/metadata 搜索相关 skill。
- progressive disclosure：先展示摘要，模型明确需要时再 Skill Loading。
- capability snapshot：每次 run 固定可见 skill 集合，避免运行中漂移。

## 边界

- [[harness/skills/skill-loading/index|Skill Loading]] 负责加载完整说明；Skill Discovery 只负责发现和选择。
- [[harness/tooling/extension-registry/index|Extension Registry]] 可以提供 skill catalog 来源，但 discovery 还包括 prompt/selection 策略。
- [[harness/tooling/tool-availability/index|Tool Availability]] 管 tools，Skill Discovery 管 task knowledge package；两者可能互相影响。

## 目录

- [[harness/skills/skill-discovery/implementations/uxarts-agent|uxarts-agent 实现]]
