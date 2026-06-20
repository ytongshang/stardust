---
title: uxarts-agent — Workspace Merge
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/subagent/merge_artifacts_tool.py
tags:
  - uxarts-agent
  - workspace-merge
---

# uxarts-agent — Workspace Merge

uxarts-agent 的 background sub-agent 在独立 workspace 中工作，主 agent 可通过 merge artifacts 工具把子 agent 产物合并回来。

特点：

- 主 agent 明确触发合并。
- 合并可以保留冲突标记，让主 agent 继续处理。
- 子 agent 写入不直接污染主工作区。

这比同步 subagent 返回文本更接近真实多 agent 协作。
