---
title: uxarts-agent — Evals
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/tests
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/e2e
tags:
  - uxarts-agent
  - evals
---

# uxarts-agent — Evals

uxarts-agent 已有 E2E 工具和若干单测，但 harness 级 eval 还没有系统化。

已有相关：

- `run_e2e_test` tool。
- tool args tests。
- sub-agent tests。
- plan mode tests。
- prompt builder / capability context tests。

可补强方向：

- tool profile ablation。
- compaction quality eval。
- pending/resume recovery eval。
- sub-agent fanout/merge eval。
- trace replay based regression。
