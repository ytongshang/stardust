---
title: uxarts-agent — Artifacts
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/codebase/codebase.py
  - uxarts-agent/uxarts/web/modules/getui/agent/types/getui_context.py
tags:
  - uxarts-agent
  - artifacts
---

# uxarts-agent — Artifacts

uxarts-agent 用 `Codebase.artifacts` 保存工作区产物的内存视图。文件工具写入后，`TextEditor` observer 会更新 artifacts，并通过 `save_attachment` 持久化到 uxa-center。

Artifacts 包含：

- 代码文件。
- 用户文档。
- design system / svg resources。
- todos/features/thoughts/reasoning steps。
- `.superun/memory` 和 `.superun/skills` 这类 agent 可读资源。

Artifacts 不是临时文件，它们与产品预览、发布、历史快照和 sub-agent merge 绑定。
