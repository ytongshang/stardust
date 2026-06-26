---
title: Meta-Harness Context Injection
type: implementation
created: 2026-06-21
updated: 2026-06-21
sources:
  - https://github.com/stanford-iris-lab/meta-harness-tbench2-artifact
  - https://github.com/stanford-iris-lab/meta-harness-tbench2-artifact/blob/main/agent.py
tags:
  - agent-harness
  - context
  - implementation
---

# Meta-Harness Context Injection

Meta-Harness 是 Terminal-Bench 2.0 的 agent scaffold。它不是新的完整 runtime，而是在 Harbor / Terminus2 agent 上加了一个很明确的 Context Injection pattern：first-turn environment snapshot injection。

## 属于哪个格子

- timing：first turn。
- source：sandbox runtime state。
- freshness：run-start snapshot，一次性探测，不每轮刷新。
- visibility：当前 benchmark agent。
- shape：追加到 initial prompt 的文本 block。
- cache boundary：不适合作为长期 memory；只对当前 sandbox run 有效。

## 它做了什么

`AgentHarness._gather_env_snapshot()` 在 agent loop 开始前执行一个 compound shell command，探测：

- working directory；
- `/app/` 文件列表；
- Python、GCC、Node、Java、Rust、Go 等工具链；
- pip / apt-get 等 package managers；
- memory summary。

随后 `_run_agent_loop()` 在首轮 prompt 前调用 `_gather_env_snapshot()`，如果拿到 snapshot，就把它追加成 `[Environment Snapshot]` block。失败时静默跳过，不破坏 agent loop。

## 为什么有用

Terminal agent 常常在前几轮花 token 和时间做机械探索：`pwd`、`ls`、`which python3`、检查 package manager。Meta-Harness 把这些低价值探索从模型行为改成 harness 的确定性 preflight step，减少早期 turn 数，也让第一轮 plan 更贴近真实环境。

这个 pattern 的关键不在 prompt 文案，而在 injection policy：

- context 来自 runtime 主动探测，不来自 user input；
- 只在 first turn 注入，避免每轮重复；
- snapshot 有明确 scope，主要服务 sandbox coding task；
- 如果探测失败，runtime 退回普通 agent 行为。

## 和 uxarts-agent 对比

uxarts-agent 已有 first-turn static context、per-turn context、post-compact context 等机制。Meta-Harness 的差异是它把 sandbox environment probe 作为 first-turn context source 显式建模：先由 harness 做一次确定性环境探测，再交给模型行动。

这可以作为 uxarts-agent 的一个可迁移小 pattern：对代码执行型任务，run start 时生成轻量 `environment snapshot`，作为 Context Injection 的 first-turn runtime block，而不是让模型自己用工具探索基本环境。
