---
title: uxarts-agent — Sandbox
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/code/run_code_tool.py
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/sandbox
tags:
  - uxarts-agent
  - sandbox
---

# uxarts-agent — Sandbox

uxarts-agent 通过 run_code、deploy、E2E 等工具与 sandbox/执行环境交互。

已有能力：

- `run_code` 在受控环境里执行代码。
- E2E 工具基于 workspace 运行测试。
- deploy/preview 与产品发布链路连接。
- sandbox session 信息由 `GetuiContext` 和相关工具共享。

当前 sandbox policy 没有完全抽成独立层，权限主要通过工具可用性、secrets/integrations 和具体工具实现约束。
