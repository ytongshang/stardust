---
title: uxarts-agent — Snapshots
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxa-center
  - uxarts-agent/uxarts/web/modules/getui/codebase/codebase_loader.py
tags:
  - uxarts-agent
  - snapshots
---

# uxarts-agent — Snapshots

uxarts-agent 通过 center 的 attachment/snapshot 体系恢复工作区历史。round 视图由快照、增量和 tombstone 共同构成。

相关行为：

- `CodebaseLoader` 加载历史产物。
- center 维护 attachment 索引和内容存储。
- 删除文件需要同步 tombstone / snapshot 视图。
- sub-agent 派生工作区时依赖父工作区状态。

当前 snapshot 主要在 center 和 codebase loader 侧实现，Python runner 只消费结果。
