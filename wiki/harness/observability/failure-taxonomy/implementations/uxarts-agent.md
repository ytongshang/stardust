---
title: uxarts-agent — Failure Taxonomy
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/core/exception
  - uxarts-agent/uxarts/web/modules/getui/exception
tags:
  - uxarts-agent
  - failure-taxonomy
---

# uxarts-agent — Failure Taxonomy

uxarts-agent 已有异常映射，但还没有完整的 harness failure taxonomy。

已有分类来源：

- model/content policy exceptions。
- tool call JSON parse / validation errors。
- user interrupted。
- insufficient credits。
- session not exists。
- timeout / heartbeat retry。
- fake tool call/output report。

后续可以把这些和 `RunItem` trace 关联，用于自动根因分析。
