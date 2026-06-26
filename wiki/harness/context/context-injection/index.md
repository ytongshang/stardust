---
title: Context Injection
type: concept
created: 2026-06-20
updated: 2026-06-21
tags:
  - agent-harness
  - context
---

# Context Injection

Context Injection 决定哪些上下文在什么时机注入给模型。关键不是具体文案，而是注入时机、来源、缓存边界和是否随 agent 类型变化。

## 它解决什么问题

Agent 需要的 context 不只来自 user input，还来自 workspace、memory、retrieval、artifact、previous run、system state 和 external services。Context Injection 负责决定这些 context 何时进入模型，以及进入后是否应该持续、刷新或移除。

它需要控制：

- 注入时机：first turn、every turn、after compaction、after tool result、sub-agent start。
- 注入来源：workspace、memory、retrieval、product state、runtime state。
- 注入范围：只给当前 agent、给 child agent、给后续所有 turns，还是只给当前 turn。
- 注入形态：system block、developer block、user-adjacent content、tool metadata。
- 缓存边界：哪些 context 稳定可缓存，哪些每轮变化。

## 要拆开的维度

很多早期实现会把 context 都写进同一个 prompt section：项目背景、memory、tool policy、当前文件变化、runtime state、workflow reminder 混在一起。这种写法短期能工作，但后续很难判断一个 block 应该首轮注入、每轮刷新、压缩后恢复，还是只给某类 sub-agent。

拆分时至少看这些维度：

| 维度 | 要回答的问题 | 例子 |
|---|---|---|
| timing | 什么时候进入模型？ | first turn、every turn、after compaction、sub-agent start |
| source | context 从哪里来？ | workspace、memory、capability catalog、sandbox probe、tool result |
| freshness | 内容是 static 还是 live？ | repo rules 很稳定；file changes 和 secrets availability 每轮可能变 |
| visibility | 哪个 agent 看得到？ | main agent、general-purpose sub-agent、read-only sub-agent |
| shape | 作为什么 message/block 进入？ | system block、developer block、user-adjacent context、tool metadata |
| cache boundary | 能不能复用或缓存？ | static profile context 可缓存；runtime state 不应长期缓存 |
| provenance | 能不能解释为什么模型看到了它？ | Event Log / trace 记录 provider、版本、刷新原因 |

这样可以把“原来写在一起”的 context 拆成一组可组合的 injection blocks，而不是继续扩大一个万能 prompt。

## 常见 injection blocks

- first-turn static context：agent profile、长期规则、稳定 workspace 背景、capability/skill snapshot。
- first-turn runtime snapshot：run 开始时探测到的 sandbox / environment state，例如工作目录、文件列表、语言工具链和 package managers。
- every-turn live context：当前 file changes、active secrets/integrations、UI/runtime state、pending approvals。
- after-compaction recovery context：压缩后必须恢复的约束、profile、tool policy、workspace summary。
- after-tool-result follow-up context：工具结果触发的补充说明、错误恢复提示、可继续执行的状态。
- sub-agent start context：对子 agent 的 self-contained brief、权限边界、workspace snapshot、可见 tools。

## 常见做法

好的实现通常会把 context 做成 provider 或 block：

- 每个 provider 声明 source、priority、refresh policy、budget cost 和 visibility。
- runtime 根据 agent profile、task state、workspace state 和 token budget 选择 context blocks。
- 注入结果记录到 Event Log 或 trace，便于解释模型为什么看到了某些信息。
- after compaction 时重新注入必要 static context，避免 summary 丢失关键约束。
- provider 不只返回文案，还应声明 timing、source、freshness、visibility、cache policy 和 budget cost。

## 目录

- [[harness/context/context-injection/implementations/uxarts-agent|uxarts-agent 实现]]
- [[harness/context/context-injection/implementations/meta-harness|Meta-Harness 实现]]
- [[harness/context/context-injection/implementations/agentspace|AgentSpace 实现]]

## 边界

- [[harness/context/input-transform/index|Input Transform]] 来自用户输入；Context Injection 是 runtime 主动补上下文。
- [[harness/context/prompt-assembly/index|Prompt Assembly]] 负责最终排序和渲染；Context Injection 负责选择和时机。
- [[harness/memory/memory-retrieval/index|Memory Retrieval]] 是一种 context source，不等于完整的 injection policy。

## 后续收集

外部项目如果显式建模 context injection timing、context block provider、cache-safe context 或 per-agent injection policy，可以在本目录下新增笔记。
