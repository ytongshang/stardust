---
title: Context Injection
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/task/prototype/prototype_generate_user_input_creator.py
  - uxarts-agent/uxarts/web/modules/getui/task/async_sub_agent/general_purpose.py
  - uxarts-agent/uxarts/web/modules/getui/task/conversation_context_window.py
tags:
  - agent-harness
  - context
---

# Context Injection

## 这个概念是什么

Context Injection 决定“哪些上下文，在什么时机，作为什么 message block 注入给模型”。它应该和业务 stage 解耦。

通用时机：

- first turn
- every turn
- after compact
- sub-agent start

通用内容：

- project structure
- memory
- wiki memory
- capability/plugin/skill snapshot
- file changes
- secrets/integrations
- runtime reminders
- workflow reminders

## uxarts-agent 现在做什么

现有注入分布在 user input creator、model init message、compact handler、sub-agent run spec 中。

主要类型：

- static context: 首轮或压缩后注入。
- per-turn context: 每轮注入。
- model init messages: 首轮初始化。
- post-compact context: 压缩边界后重新注入。
- business reminders: 例如 architecture planning、README sync、language rule。

## uxarts-agent 怎么做

prototype：

- `_build_context_reminder(context, include_static)`
- `build_static_context(context)`
- `_build_per_turn_context(context)`

static context 包括：

- capability static context。
- `memory/MEMORY.md` 或 memory directory 状态。
- `.superun/memory/` wiki memory 概览。

per-turn context 包括：

- capability runtime state。
- plugins/secrets。
- base_h5_url。
- miniprogram。
- file_changes。
- architecture planning 每轮提醒。

sub-agent：

- `SubAgentRunSpec.post_compact_context_blocks` 决定压缩后是否重新注入 static context。
- general-purpose sub-agent 使用 `build_static_context` 和 `PerTurnContextBlocks.common_blocks`。
- read-only sub-agent 不注入 heavy static/per-turn context。

## 不能丢的细节

- `stage` 只是业务条件，不是注入层级。
- “首轮注入”和“压缩后重新注入”是通用时机。
- architecture planning 文案是业务 block，不是 runtime API。
- capability context 要避免每次 LLM call 重复塞全量 catalog。
- compressed 是粘性的，但不能让 first-turn static context 每轮重复注入。

## 建议抽象

```python
@dataclass
class ContextInjectionPolicy:
    on_first_turn: list[ContextBlockProvider]
    on_every_turn: list[ContextBlockProvider]
    on_after_compact: list[ContextBlockProvider]
    on_sub_agent_start: list[ContextBlockProvider]
```

## Related Implementations

- 后续收集外部项目时，重点看它有没有显式的 injection timing，而不是只看 prompt 文案。
