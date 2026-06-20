---
title: uxarts-agent — Remote Tools
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/plugin/plugin_manifest_tool_runtime.py
tags:
  - uxarts-agent
  - remote-tools
---

# uxarts-agent — Remote Tools

uxarts-agent 通过 plugin manifest 动态合并远程工具：

- `build_plugin_manifest_tools` 根据 session 的 plugin catalog/state 构造工具。
- remote tool 和静态工具重名时跳过并记录 warning。
- sub-agent 调用远程工具时使用 main session id 解析 plugin 配置和凭证。
- SUPERUN_CLOUD 相关工具会根据 stage/sub-agent 权限裁剪。

这和 MCP/remote tool registry 属于同一类问题，但当前实现是产品 plugin manifest 体系。
