---
title: Skill Evaluation
type: concept
created: 2026-06-22
updated: 2026-06-22
tags:
  - agent-harness
  - skills
  - evals
---

# Skill Evaluation

Skill Evaluation 衡量一个 skill 是否真的改变了 agent 行为。它关心的不是 agent 最终有没有完成任务，而是：skill 被发现、加载或明确提供后，agent 是否按 skill 编码的 workflow、tool choice、naming convention、artifact schema 或禁止事项行动。

## 它解决什么问题

很多 skill 的价值不是让 agent 第一次“会做”，而是让它按我们希望的方式做。只看 final answer 容易漏掉这些差异：agent 可能不用 skill 也能完成任务，但会用错 CLI、跳过安全检查、输出不符合 schema，或者没有遵循团队约定。

它需要回答：

- skill 是否提高 task success。
- skill 是否改变 instruction following。
- skill 是否让小模型接近大模型。
- skill 条款里哪些真的影响了行为，哪些只是额外 context。
- 失败是 skill 本身无效，还是 discovery / loading / context budget 失败。

## 常见做法

常见 skill eval 会做 paired conditions：

- `without skill`：没有 skill，只给任务和环境。
- `with selected skill`：明确提供目标 skill，测试 utility-after-selection。
- `with skill catalog`：只提供 catalog / discovery 入口，测试 discovery + loading。

评估时最好拆开：

- `goal completion`：产物是否正确、任务是否完成。
- `instruction following`：是否遵循 skill 的 workflow、tool、命名、格式和禁止项。
- per-rubric delta：哪些具体条款改变了行为。

## 边界

- [[harness/skills/skill-discovery/index|Skill Discovery]] 决定 agent 能不能找到 skill。
- [[harness/skills/skill-loading/index|Skill Loading]] 决定 skill 内容是否被正确加载进上下文。
- [[harness/observability/evals/index|Evals]] 提供通用任务、metric、trace 和 judge 机制；Skill Evaluation 把这些机制应用到 skill utility 上。

## 目录

- [[harness/skills/skill-evaluation/implementations/agentic-skills-evaluation|Agentic Skills Evaluation 实现]]
