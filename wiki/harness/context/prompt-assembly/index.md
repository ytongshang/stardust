---
title: Prompt Assembly
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - context
  - prompt
---

# Prompt Assembly

Prompt Assembly 是把 agent 的长期身份、任务目标、运行规则、工具使用策略、应用上下文、模型适配要求和当轮动态输入，编译成最终发给模型的 prompt / messages 的过程。

它不是简单的字符串拼接，而是一个 prompt 编译层：上游把用户输入、检索上下文、workspace 状态、记忆、agent profile 和 runtime state 交给它；它负责按稳定结构、优先级和模型约束组织成模型可消费的上下文。

## 它解决什么问题

Agent harness 里 prompt 往往会逐渐散落在 agent 初始化、工具定义、上下文注入、压缩恢复、sub-agent 创建和模型适配代码中。Prompt Assembly 把这些分散规则收束成一个可检查的生成过程，解决几个问题：

- 让 agent 的行为约束有稳定入口，而不是散在多个 if/else 里。
- 让不同 agent profile、任务状态、模型和工具集合可以复用同一套组装机制。
- 让 prompt 的变化可追踪、可 diff、可测试、可回放。
- 让静态 prompt、动态上下文和当轮输入有明确边界，避免互相覆盖。
- 让缓存稳定的 prompt block 和每轮变化的 context block 分开，方便控制成本和延迟。

## 常见做法

Prompt Assembly 通常会把最终 prompt 拆成一组有顺序、有来源、有适用条件的 blocks：

- identity / role：agent 是谁，承担什么职责。
- task goal / workflow：当前任务目标、执行流程、完成标准。
- behavior rules：语言、风格、安全边界、代码规范、应用约束。
- tool policy：什么时候用工具、工具调用约束、失败恢复规则。
- context blocks：项目状态、文件、artifact、memory、检索内容、应用参数。
- model adapter blocks：不同模型的 tool schema、message shape、reasoning hints、缓存边界。
- runtime reminders：本轮或压缩恢复后的临时约束。

组装时通常需要做几类决策：

- block selection：当前 agent profile、task state、model、workspace 状态下哪些 block 生效。
- ordering：哪些规则必须靠前，哪些上下文可以靠后，哪些必须贴近 user input。
- precedence：system rule、application rule、user request、tool result 冲突时谁覆盖谁。
- rendering：同一个逻辑 block 如何渲染成 system message、developer message、user message 或 tool-specific metadata。
- budgeting：超出 context window 时哪些 block 可以裁剪、摘要或延后注入。
- observability：记录最终 prompt 的结构化来源，便于 trace、debug 和 eval。

## 为什么值得单独建模

Prompt Assembly 单独建模的价值在于，它把“agent 应该如何表现”从“某次请求的上下文是什么”里拆出来。这样 context 可以变化，工具可以变化，模型可以变化，但 agent 的核心行为约束和 prompt 结构仍然可控。

它的优势包括：

- 可维护性：新增 agent profile、任务状态、工具策略或模型适配时，不需要到处改 prompt 字符串。
- 可复用性：不同 agent 可以共享 block registry，只替换 profile 和应用 block。
- 可测试性：可以对某个 profile 的最终 prompt 做 snapshot、lint、eval 和 replay。
- 可解释性：线上行为异常时，可以看到每条规则来自哪个 block，而不是只能看一大段文本。
- 可迁移性：切换模型或消息格式时，主要改 adapter/render 层，而不是重写应用 prompt。

## 和相邻概念的边界

- [[harness/context/input-transform/index|Input Transform]] 负责把产品原始输入转成 LLM 可消费的 user content；Prompt Assembly 决定这些 content 放在最终消息结构里的什么位置。
- [[harness/context/context-injection/index|Context Injection]] 负责哪些上下文在什么时机进入运行；Prompt Assembly 负责进入后的组织、排序和渲染。
- [[harness/context/context-budgeting/index|Context Budgeting]] 负责预算判断和裁剪策略；Prompt Assembly 应该尊重预算结果，并暴露哪些 block 可裁剪。
- [[harness/context/compaction/index|Compaction]] 负责历史摘要和恢复；Prompt Assembly 负责把 compact summary、恢复提醒和当前任务重新拼回稳定 prompt 结构。

## 判断外部实现是否值得记录

值得记录的实现一般不只是“拼了一段 system prompt”，而是有明确的 prompt block、profile、policy、adapter、snapshot/eval、cache-stable layout 或可回放机制。

如果外部项目只是把 role、rules、tool tips 写在一个长字符串里，通常只需要简短记为已有类似概念；如果它把 prompt assembly 做成可组合、可测试、可随 agent profile 编译的层，就值得写实现笔记。

## 目录

- [[harness/context/prompt-assembly/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部项目如果有 prompt block registry、profile-based prompt assembly、cache-stable prompt layout 或 prompt eval，可以放到本目录。
