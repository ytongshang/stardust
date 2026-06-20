---
title: uxarts-agent — Memory Retrieval
type: implementation
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/prototype/prototype_generate_user_input_creator.py
tags:
  - uxarts-agent
  - memory-retrieval
---

# uxarts-agent — Memory Retrieval

uxarts-agent 当前 memory retrieval 偏“文件系统提示 + 模型主动读取”：

- `build_static_context` 注入 `memory/MEMORY.md` 摘要或提示 memory 目录存在。
- wiki memory 注入 hot/index 概览，并提示模型在遇到陌生引用时先搜索 `.superun/memory/`。
- 模型使用 file tools 读取具体记忆文件。

当前没有独立向量检索层；更多依赖目录结构、摘要和模型主动探索。
