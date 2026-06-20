---
title: Context Budgeting
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - context
  - budgeting
---

# Context Budgeting

Context Budgeting 是对 token、图片、多模态内容、工具输出和缓存前缀的预算管理。它决定什么内容进入模型、进入多少、是否需要截断或压缩。

## 它解决什么问题

Context window 不是无限资源。没有 Context Budgeting，agent 会在历史、tool output、workspace 文件、memory、图片和 retrieval 结果之间随机挤占空间，导致关键约束丢失、成本失控或模型输入超限。

它需要回答：

- 每类 context 的预算上限是多少。
- 哪些内容必须保留，哪些可以裁剪、摘要、链接化或延后注入。
- tool result 太大时，如何保留可用 observation 而不是整段塞回模型。
- prompt caching 的 stable prefix 如何保持稳定。
- multimodal input 的图片数量、分辨率和 token cost 如何控制。

## 常见做法

常见策略包括：

- priority budget：给 system rules、current task、recent events、retrieval、tool results 设置不同优先级。
- lossy transform：对长 tool output、文件内容、搜索结果做摘要或 top-k 选择。
- offloading：把完整内容保存到 artifact/workspace，只把引用和摘要给模型。
- adaptive compaction：接近预算时触发 [[harness/context/compaction/index|Compaction]]。
- accounting：记录每个 block 的 estimated tokens、actual usage 和裁剪原因。

## 边界

- Context Budgeting 决定“能放多少、舍弃什么”；[[harness/context/prompt-assembly/index|Prompt Assembly]] 决定“放在哪里、怎么渲染”。
- [[harness/context/compaction/index|Compaction]] 是一种预算不足后的历史压缩机制，不是全部 budgeting。
- [[harness/tooling/tool-results/index|Tool Results]] 可以提供大结果的 envelope 和 refs，Budgeting 决定哪些 result 进入下一轮。

## 目录

- [[harness/context/context-budgeting/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

关注外部项目的 token accounting、image budgeting、tool output offloading、prompt caching layout 和 context prioritization。
