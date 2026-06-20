---
title: uxarts-agent — Skill Materialization
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/plugin/capability_brief_injector.py
tags:
  - uxarts-agent
  - skill-materialization
---

# uxarts-agent — Skill Materialization

uxarts-agent 会把 skill materialize 到工作区：

```text
.superun/skills/<id>/SKILL.md
```

模型通过文件系统读取具体 skill 内容。这样 skill 既是能力说明，也是 workspace 中可检索、可版本化的资源。

sub-agent 可以按自己的 session/workspace 继承或安装 skill。
