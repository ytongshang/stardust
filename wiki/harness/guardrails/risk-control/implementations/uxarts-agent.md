---
title: uxarts-agent — Risk Control
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent
  - uxa-center
tags:
  - uxarts-agent
  - risk-control
---

# uxarts-agent — Risk Control

uxarts-agent / uxa-center 的 risk control 不是单一 harness 模块，主要体现在：

- content policy violation 映射和 round_extra 更新。
- save attachment 的文本风控。
- token/credit 检查。
- 用户中断、余额不足、session 不存在等异常类型直接抛出。

这类机制更偏产品安全和商业状态，但对 agent harness 的可控性很重要。
