---
title: Agentic Harness Engineering — Evals
type: implementation
created: 2026-06-21
updated: 2026-06-21
sources:
  - https://github.com/china-qijizhifeng/agentic-harness-engineering
  - https://github.com/china-qijizhifeng/agentic-harness-engineering/blob/main/README.md
  - /Users/rancune/Work/agent/automations/agent-harness-radar/2026-06-21/china-qijizhifeng__agentic-harness-engineering
  - agentic_harness_engineering.pdf
  - evolve.py
  - agents/evolve_agent/evolve_prompt.md
  - experiments/evolved_harness/
tags:
  - agentic-harness-engineering
  - evals
  - observability
  - self-evolution
---

# Agentic Harness Engineering — Evals

AHE 把 harness 改动变成可评估、可归因、可回滚的实验循环。它的重点不是让 model 变强，而是在 base model 固定时，让 system prompt、tool description、tool implementation、middleware、skills、sub-agents、long-term memory 这些外部 harness 组件自己积累经验。

这篇放在 [[harness/observability/evals/index|Evals]] 下，而不是放在 Prompt Assembly 或 Tool Execution 下，是因为它最有价值的不是某个单点组件，而是“如何判断 harness 改动是否真的有效”的外层协议。

## Effect Definition

AHE 对“效果”的定义很窄：

- primary metric 是 `pass@1`，也就是单次尝试成功率。
- `pass@k` 只作为能力上限和诊断信号，不是优化目标。
- timeout 和 infrastructure-aborted trial 在 `pass@1` 下按失败处理。
- 每轮对比上一轮的 task-level transition：`fail -> pass` 是 flipped，`pass -> fail` 是 regression。
- partial-pass task 是高价值对象：同一个 task 有的 rollout 过、有的不过，说明成功策略已经存在但不稳定。

这比“总体分数提高”更适合 harness evolution，因为它能回答某个具体改动到底让哪些任务翻转，哪些任务回归。

## Evaluation Data Model

AHE 的 eval 可以拆成四个对象：

| Object | 来源 | 作用 |
|---|---|---|
| `TaskResult` | verifier 的 `reward.txt` / exception / timeout | 把每个 rollout 归成 pass、fail、exception |
| `IterationStats` | 当前轮所有 task results | 计算 pass rate、`pass@1`、`pass@k`、timeout、exception |
| `IterationDiff` | 当前轮 vs 上一轮 | 计算 `flipped`、`regressed`、`stable_pass`、`stable_fail` |
| `ChangeEvaluation` | 上一轮 manifest + 当前轮 diff | 给每个 change 判定 effective、partial、mixed、ineffective、harmful |

关键是 `IterationDiff`。AHE 不只问“这一轮分数是多少”，而是问：

- 上轮失败、这轮通过的 task 是哪些？
- 上轮通过、这轮失败的 task 是哪些？
- 有多个 rollout 时，哪些 task 从 0/2 变成 1/2，虽然还没完全通过？
- regression 是否来自已声明的 `risk_tasks`，还是完全没预见？

## Evolution Loop

AHE 的一轮不是 prompt self-improvement，而是一个外层实验协议：

1. `Evaluate`：用当前 harness 跑 benchmark，保存 raw trace、runtime log 和 verifier reward。
2. `Analyze`：Agent Debugger 把多任务轨迹压成 `analysis/overview.md` 和 per-task detail。
3. `Improve`：Evolve Agent 只允许写 `workspace/` 内的 harness 组件。
4. `Declare`：每个 logical change 必须写入 `change_manifest.json`，包括 `predicted_fixes` 和 `risk_tasks`。
5. `Verify`：下一轮 eval 读取上一轮 manifest，用实际 flipped/regressed task 给每个 change 打 verdict。

这个 staggered design 很关键：本轮写出的 harness 不是本轮证明自己，而是下一轮用 task delta 证伪自己。

## Change Manifest Contract

`change_manifest.json` 是 AHE 里最值得借鉴的部分。它把每个 harness edit 变成一个可证伪的合同，而不是事后解释。

最小字段应该包括：

```json
{
  "iteration": 3,
  "changes": [
    {
      "id": "chg-1",
      "type": "new|improvement|rollback",
      "description": "What changed and why",
      "files": ["middleware/execution_risk_hints.py"],
      "failure_pattern": "Agent passes final check, then cleanup resets published state",
      "predicted_fixes": ["configure-git-webserver", "git-multibranch"],
      "risk_tasks": ["long-horizon-task-a"],
      "constraint_level": "middleware|tool_impl|tool_desc|skill|prompt",
      "why_this_component": "Prompt reminders were ignored; tool/middleware can intercept the behavior."
    }
  ]
}
```

下一轮 eval 后，系统拿 `predicted_fixes` 和真实 `flipped` 相交，拿 `risk_tasks` 和真实 `regressed` 相交：

- predicted task 全部 flipped，且没有风险命中：`EFFECTIVE`
- 只有部分 predicted task flipped：`PARTIALLY_EFFECTIVE`
- 同时修好一些 task 又打坏 risk task：`MIXED`
- 没修好 predicted task：`INEFFECTIVE`
- risk 命中且没有修好东西：`HARMFUL`

这里的判定不是严格 causal proof，只是把“修改声明”和“下一轮结果”对齐。它的价值在于让自我进化不能只靠 narrative 自证。

## Observability Layers

AHE 把可观测性拆成三层：

- component observability：每类 harness component 都是 file-level surface，便于 diff、commit、rollback。
- experience observability：trace 先被压成 layered evidence corpus，Evolve Agent 默认读分析报告，必要时 drill down 到原始 trace。
- decision observability：change manifest 把每个 edit 变成 falsifiable contract。

这给我们的启发是：harness eval 不只需要 benchmark runner，还需要把“改了什么、为什么改、预计影响什么、实际发生什么”一起记录。

## Evolved Components

最终 evolved harness 里最有价值的不是大段 prose，而是可执行的约束：

- `run_shell_command.py` 增加 publish-state guard：final/evaluator-style check 通过后，把当前文件/目录/service state 标记为 publish state，阻止后续 cleanup/reset 把已通过状态破坏掉。
- `ExecutionRiskHintsMiddleware` 观察跨步骤 shell 行为，把 shallow validation、localhost-only check、dependency retry、thin benchmark margin、post-success reset 等风险提升到下一轮 model input。
- `LongTermMEMORY.md` 记录稳定经验，例如 contract surface 必须精确命中、成功后冻结发布状态、最终验证必须从提交 artifact 本身闭环。

这些组件说明：如果一个 prompt rule 反复失败，应该把它迁移成 tool-level guard、middleware reminder 或 memory，而不是继续追加更长的 prompt。

## Minimal Version For Our Eval

如果后面我们要参考 AHE 做自己的 agent eval，可以先做一个轻量版，不需要完整复刻 Terminal-Bench / Harbor / E2B：

1. 固定一组小而稳定的 regression tasks，每个 task 有明确 verifier。
2. 每次改 harness 前，写 `change_manifest.json`，声明 expected fixes 和 regression risks。
3. 每个 eval run 保存 per-task result、trace、tool calls、token/cost、wall time。
4. 每轮生成 `iteration_diff.json`：`flipped`、`regressed`、`stable_pass`、`stable_fail`。
5. 每轮生成 `change_evaluation.json`：把上一轮 manifest 和当前轮 diff 对齐。
6. dashboard 优先展示 task transitions，而不是只展示 aggregate score。

最小目录可以是：

```text
eval_runs/
  iteration_001/
    input_harness_snapshot/
    results.json
    traces/
    change_manifest.json
  iteration_002/
    results.json
    iteration_diff.json
    change_evaluation.json
```

这样我们以后调 context injection、tool execution、memory retrieval、sub-agent lifecycle 时，都能回答同一个问题：这个改动预计修什么，下一轮实际修了什么，又打坏了什么？

## Boundaries

- AHE 的归因仍然偏粗：代码里主要把 `predicted_fixes` 与下一轮 `flipped` 相交，把 `risk_tasks` 与 `regressed` 相交；它不能证明文件改动和任务翻转之间有严格因果。
- 论文里 regression prediction 明显弱于 fix prediction，说明 self-evolution 很擅长解释“为什么会好”，但不太会预见“会破坏哪里”。
- 这是 Terminal-Bench / Harbor / E2B / NexAU 绑定很深的研究原型，不等于成熟的通用 self-improvement governance。

## Source Links

- GitHub: [china-qijizhifeng/agentic-harness-engineering](https://github.com/china-qijizhifeng/agentic-harness-engineering)
- Local snapshot: `/Users/rancune/Work/agent/automations/agent-harness-radar/2026-06-21/china-qijizhifeng__agentic-harness-engineering`
- Paper PDF: `/Users/rancune/Work/agent/automations/agent-harness-radar/2026-06-21/china-qijizhifeng__agentic-harness-engineering/agentic_harness_engineering.pdf`
- Main loop: `evolve.py`
- Evolve Agent contract: `agents/evolve_agent/evolve_prompt.md`
- Final evolved harness: `experiments/evolved_harness/`

## Related

- [[harness/observability/tracing/index|Tracing]]
- [[harness/observability/failure-taxonomy/index|Failure Taxonomy]]
- [[harness/tooling/tool-execution/index|Tool Execution]]
- [[harness/memory/long-term-memory/index|Long-Term Memory]]
- [[harness/guardrails/risk-control/index|Risk Control]]
