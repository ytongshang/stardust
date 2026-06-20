---
title: uxarts-agent — Sandbox Policy
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/code/run_code_tool.py
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/sandbox
tags:
  - uxarts-agent
  - sandbox-policy
---

# uxarts-agent — Sandbox Policy

uxarts-agent 的 sandbox policy 目前分散在 run_code/deploy/E2E 等工具中：

- tool availability 决定是否暴露 run_code/deploy/E2E。
- integrations/secrets 决定执行环境凭证。
- sandbox helper 复用 session 运行态。

尚未抽成统一 sandbox policy 模块。
