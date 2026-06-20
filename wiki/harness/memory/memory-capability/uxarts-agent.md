---
title: Memory & Capability
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/agent/types/getui_context.py
  - uxarts-agent/uxarts/web/modules/getui/agent/tools/plugin/capability_brief_injector.py
  - uxarts-agent/uxarts/web/modules/getui/task/prototype/prototype_generate_user_input_creator.py
tags:
  - agent-harness
  - memory
  - capability
---

# Memory & Capability

## 这个概念是什么

Memory 是 agent 需要跨轮或跨 session 使用的知识。Capability 是当前 session 可用插件、技能、远程工具和运行状态的快照。

这两者都属于 context/capability layer，不应只散落在 prompt 里。

## uxarts-agent 现在做什么

Memory：

- project memory: `memory/MEMORY.md` 和 memory directory。
- wiki memory: `.superun/memory/`，来自用户级 wiki session。
- compact summary: 历史压缩摘要。

Capability：

- plugin catalog。
- plugin state。
- skill catalog。
- remote manifest tools。
- static capability context。
- per-turn runtime state。

## uxarts-agent 怎么做

Memory 注入：

- `build_static_context` 读取 `memory/MEMORY.md`。
- 如果 memory 目录存在但 index 缺失，会提示模型自行探索。
- wiki memory 预写到 `.superun/memory/`，并注入 hot/index 概览。
- 压缩后重新注入 static context。

Capability 注入：

- `prepare_capability_context` / `get_capability_snapshot` 拉取 snapshot。
- static context 给模型能力摘要。
- per-turn context 给 runtime state。
- skill materialize 到 `.superun/skills/<id>/SKILL.md`。
- plugin enabled 后合并 remote manifest tools。

## 不能丢的细节

- wiki memory 是只读辅助知识，不等于项目 artifacts。
- capability snapshot 与 `GetuiContext` 生命周期绑定，避免重复拉取。
- static capability context 不应每次 LLM API call 重复塞。
- sub-agent plugin dispatch 使用 main session id。
- skill 是 progressive disclosure，不是所有内容都塞进 prompt。

## Related Implementations

- 外部实现如果只有 vector memory，不算完整覆盖。
- 值得关注的是 memory recall eval、capability freshness、skill loading strategy、memory provenance。
