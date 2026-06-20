---
title: Workspace & Artifacts
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/types/getui_context.py
  - uxarts-agent/uxarts/web/modules/getui/codebase/codebase.py
  - uxarts-agent/uxarts/web/modules/getui/codebase/codebase_loader.py
tags:
  - agent-harness
  - workspace
---

# Workspace & Artifacts

## 这个概念是什么

Workspace 是 agent 可以读取、修改和发布的工作区。对产品 agent 来说，它不是普通本地文件系统，而是 artifact、attachment、snapshot、sandbox、deploy、sub-agent merge 的组合。

## uxarts-agent 现在做什么

`GetuiContext` 持有运行时工作区：

- `CodebaseLoader`
- `Codebase`
- `TextEditor`
- `TaskManager`
- design system
- svg resources
- user documents
- todos/features/thoughts/reasoning steps
- secrets/integrations/file_changes
- wiki memory overlay

## uxarts-agent 怎么做

加载：

- `CodebaseLoader.load_codebase`
- `CodebaseLoader.load_wiki_codebase`
- startup resources 加载 page resources、integrations、secrets、file_changes、reference_map、wiki memory。

写入：

- agent 通过 file tools 调 `TextEditor`。
- editor observer 更新 `Codebase.artifacts`。
- `save_attachment` 持久化到 center。
- delete/update 通过 codebase 同步 artifacts。

特殊能力：

- `.superun/memory/` wiki memory overlay。
- sub-agent 工作区复制和 merge artifacts。
- Supabase/deploy/run_code/E2E 与 workspace 绑定。
- miniprogram 文件写入后失效二维码缓存。

## 不能丢的细节

- artifact content 和 attachment backend 是产品存储，不是临时文件。
- editor policy 是写权限硬边界。
- wiki memory overlay 在 `_initialized=False` 阶段进行，避免 observer 双写。
- sub-agent 写的是自己的 workspace，回主 agent 要 merge。
- snapshots/tombstones/file_changes 是恢复和历史视图的一部分。

## 建议抽象

未来可以有 `WorkspaceBackend`，但第一步只包住现有对象：

```python
class WorkspaceBackend:
    async def load(self): ...
    async def read_file(self, path): ...
    async def write_file(self, path, content, policy): ...
    async def delete_file(self, path): ...
    async def snapshot(self): ...
    async def merge_from_sub_agent(self, agent_id): ...
```

## Related Implementations

- 外部项目如果只有 VFS，不等于覆盖产品 workspace。
- 值得关注的是 workspace isolation、merge conflict policy、artifact provenance、snapshot replay。
