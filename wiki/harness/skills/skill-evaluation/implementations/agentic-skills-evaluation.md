---
title: Agentic Skills Evaluation — Skill Evaluation
type: implementation
created: 2026-06-21
updated: 2026-06-22
sources:
  - https://arxiv.org/html/2606.17819v1
  - https://arxiv.org/abs/2606.17819
tags:
  - skills
  - skill-evaluation
  - skill-utility
  - evals
  - agentic-skills
---

# Agentic Skills Evaluation — Skill Evaluation

这篇记录的是论文 [A Framework for Evaluating Agentic Skills at Scale](https://arxiv.org/html/2606.17819v1) 的 eval 思路。它最有价值的点不是某个 skill 实现，而是提出了一个可复用的问题定义：**一个 skill 被加载后，到底有没有改变 agent 行为？**

它更适合放在 [[harness/skills/skill-evaluation/index|Skill Evaluation]] 下：核心对象是 skill 的 utility，而不是通用 eval infrastructure。它仍然会用到 [[harness/observability/evals/index|Evals]] 的任务、rubric、trace 和 judge 方法。

它和 [[harness/observability/evals/implementations/agentic-harness-engineering|Agentic Harness Engineering]] 互补：

- AHE 评估 harness edit：一个改动预计修什么，下一轮真实 task delta 是什么。
- Agentic Skills Evaluation 评估 skill utility：同一任务 with-skill / without-skill 的行为差异是什么。

## Core Question

传统 benchmark 常问“agent 能不能完成任务”。这篇把问题改成：

- agent 是否遵循 skill 中编码的 workflow、convention、library choice、naming rule、prohibited pattern？
- skill 是否提供了 base model 原本没有的知识或能力？
- skill 是提高 goal completion，还是只是改变 surface behavior？
- 更小/更便宜的 model 加上 skill 后，能不能接近更大的 model？

这对我们很重要：很多 skill 的价值不是让 agent 第一次“会做”，而是让它**按我们希望的方式做**。

## Evaluation Pipeline

论文的 pipeline 是从一个 skill 自动合成 eval task：

1. `Analyze skill / intent`：读 skill 内容，必要时结合 user intent、issue、ticket 或历史对话，推断真实使用场景。
2. `Environment engineering`：先判断任务是否可执行，需要什么环境。
3. `Task generation`：从 skill 内容生成 realistic task proposal，并准备 input artifacts。
4. `Task validation`：检查 task 是否可执行、环境是否健康、描述是否清楚、有没有泄露 rubric。
5. `Run evaluation`：solver agent 解题；judge agent 用 hidden rubrics 和 solver logs 打分。

environment engineering 这步很实用。它要求在生成 task 前先枚举 skill 的依赖：

- CLI / tool access
- MCP server
- external network / API
- auth / credential injection
- env vars
- runtime / language environment
- multi-turn evaluation support
- existing repository / code files / git state
- database
- browser / UI
- local running service
- input files
- pre-populated external service state

这个分类可以直接借给我们的 skill eval：先判断一个 skill 能不能在当前 harness 里被稳定评测，再决定是否生成 task。

## With-Skill / Without-Skill

每个 task 跑两个条件：

- `without skill`：agent 只拿到 task description 和可执行环境。
- `with skill`：agent 拿到同样 task 和环境，并且被明确告知相关 skill 可用。

二者分数差就是 `skill delta`。这个设计故意避开 skill discovery 问题，先测“skill 被选中后有没有用”。因此它适合回答 utility-after-selection，不适合回答“agent 能不能自动找到正确 skill”。

如果我们自己做 eval，最好把条件拆成三种：

1. `without skill`：没有 skill。
2. `with selected skill`：明确提供目标 skill，测试 utility。
3. `with skill catalog`：只提供 catalog / discovery 入口，测试 discovery + loading。

这样可以区分 skill 本身无效，还是 discovery / loading 失败。

## Rubric Design

论文默认生成两套 hidden rubrics，每套总分 100：

| Rubric | 衡量什么 | 例子 |
|---|---|---|
| `goal completion` | 产物是否正确、任务是否完成 | 文件存在、输出正确、脚本能跑 |
| `instruction following` | 是否遵循 skill 的 workflow / convention | 使用指定 CLI、禁止旧命令、遵守命名和结构 |

这个拆分很重要。一个 agent 可能不用 skill 也能做完任务，但做法不符合 skill 的偏好；这时 `goal completion` 高，`instruction following` 低。反过来，一个 skill 可能只改变风格，不提高最终产物正确性，也能通过这个拆分看出来。

论文里的代表例子是 Hugging Face `hf-cli` skill：without-skill 时 agent 倾向使用旧的 `huggingface-cli`，with-skill 后会切到新的 `hf` 命令。rubric 不只问脚本能不能跑，还明确检查：

- 是否完全不出现 `huggingface-cli`
- 是否用 `hf auth whoami`
- 是否用 `hf upload`
- 是否用正确的 `hf models list`

这说明好的 skill eval 应该评估“具体行为是否被改变”，而不是只给一个总分。

## What The Results Suggest

论文的大规模实验用了约 500 个真实 open-source skills 和约 1000 个生成任务，比较 19 个 agent-model configurations。主要结论：

- skill 的收益主要体现在 `instruction following`，不是单纯 goal completion。
- 较小模型从 skill 中获益通常更明显；skill 可以缩小小模型和大模型的差距。
- workflow 型 skill 最有效，例如 media/file processing、security/compliance，因为它们有明确步骤、CLI、格式和 report schema。
- 泛泛 best-practice 型 skill 收益较小，因为它们更像 guideline，不提供可执行流程。

这给我们的启发是：如果一个知识能写成 workflow、checklist、CLI sequence、artifact schema，它更适合做 skill；如果只是抽象原则，做成 skill 后也可能很难在 eval 中体现稳定收益。

## Minimal Version For Our Skill Eval

后面我们可以先做轻量版：

```text
skill_eval_runs/
  <skill_id>/
    tasks/
      task_001/
        task.md
        inputs/
        rubrics.goal.json
        rubrics.instruction.json
    runs/
      without_skill/
        trace.json
        artifacts/
        scores.json
      with_selected_skill/
        trace.json
        artifacts/
        scores.json
      with_skill_catalog/
        trace.json
        artifacts/
        scores.json
    skill_delta.json
```

最小指标：

- `goal_completion_score`
- `instruction_following_score`
- `overall_score`
- `skill_delta`
- `cost`
- `tokens`
- `wall_time`
- per-rubric item delta

最小流程：

1. 选择一个 skill。
2. 人工或半自动生成 3-5 个 realistic tasks。
3. 每个 task 写 hidden rubrics，至少拆 goal completion 和 instruction following。
4. 跑 without-skill / with-selected-skill。
5. 可选跑 with-skill-catalog，用来测试 discovery。
6. 看 per-rubric delta，判断 skill 哪些条款真的改变了行为。

## Boundaries

- 论文的 with-skill 条件明确告诉 agent 相关 skill 可用，所以不测 discovery。
- 评分依赖 LLM-as-judge；rubric 再具体，也仍然可能有 judge bias。
- 自动构造任务时必须防 rubric leakage，否则 solver 可能从 task description 里直接读到要被评的行为。
- 论文过滤掉了很多难稳定复现的 skill 环境，例如 DB、MCP、多轮交互、pre-populated external state；真实产品 skill eval 会更复杂。

## Related

- [[harness/skills/index|Skills]]
- [[harness/skills/skill-evaluation/index|Skill Evaluation]]
- [[harness/skills/skill-discovery/index|Skill Discovery]]
- [[harness/skills/skill-loading/index|Skill Loading]]
- [[harness/skills/skill-materialization/index|Skill Materialization]]
- [[harness/observability/tracing/index|Tracing]]
- [[harness/observability/failure-taxonomy/index|Failure Taxonomy]]
