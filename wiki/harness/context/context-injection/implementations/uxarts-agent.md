---
title: Context Injection
type: concept
created: 2026-06-20
updated: 2026-06-21
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

## 原来写在一起的问题

早期实现里，很多 context 是通过同一条 user-adjacent reminder 或同一个 context builder 拼进去的。这样会把几类不同生命周期的内容混在一起：

- first-turn static context：capability static context、memory directory 状态、wiki memory 概览。
- every-turn live context：capability runtime state、plugins/secrets、base_h5_url、miniprogram、file_changes。
- business reminders：architecture planning、README sync、language rule。
- recovery context：compact 后要重新注入的 static blocks。

这些内容看起来都叫 context，但刷新策略不同。static context 适合首轮或 after compact；live context 需要每轮刷新；business reminders 更像 task policy block；recovery context 只在压缩边界后补回来。把它们写在一起，会导致两个风险：

- 重复注入：例如 compressed 状态导致 first-turn static context 后续每轮都被当作新 context。
- 错误可见性：某些 heavy context 或权限相关信息不应该给 read-only sub-agent 或所有 child agent。

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

更细一点，`ContextBlockProvider` 不应该只是返回一段字符串，还应该声明：

```python
@dataclass
class ContextBlockProvider:
    source: ContextSource
    timing: set[InjectionTiming]
    freshness: Literal["static", "run_snapshot", "per_turn", "event_triggered"]
    visibility: set[AgentVisibility]
    cache_policy: CachePolicy
    budget_cost: int | None

    async def build(self, context: RunContext) -> ContextBlock: ...
```

这样 `project structure`、`memory`、`capability snapshot`、`file changes`、`runtime reminders` 不再靠“写在一起”的 prompt 顺序区分，而是靠 provider metadata 决定何时出现、给谁看、是否缓存、是否在 compact 后恢复。

## Related Implementations

- 后续收集外部项目时，重点看它有没有显式的 injection timing，而不是只看 prompt 文案。
