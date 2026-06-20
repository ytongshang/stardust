---
title: Tool Results
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - tooling
---

# Tool Results

Tool Results 是工具执行后回到模型和产品系统的 observation/result envelope。它不仅是字符串，还可能包含结构化消息、附件、错误标记和展示信息。

## 它解决什么问题

Tool result 既要给模型继续推理，也要给产品层展示、给 Event Log 保存、给 replay/eval 使用。只返回一段字符串会丢掉状态、来源、错误类型、artifact refs 和可展示信息。

Tool Results 需要表达：

- success / error / interrupted / pending 等状态。
- 给模型看的 concise observation。
- 给 UI 或产品层看的 display payload。
- 完整数据的 artifact refs 或 storage refs。
- 和原始 tool call 的 call id / correlation id。

## 常见做法

常见 result envelope 会区分：

- model content：短文本、structured observation、下一步提示。
- system metadata：duration、status、retryable、error type、truncated flag。
- artifact refs：文件、图片、网页、diff、report、dataset。
- display blocks：表格、图片、附件、progress update。
- raw payload：只给系统保存，不直接放进 model context。

## 边界

- [[harness/tooling/tool-execution/index|Tool Execution]] 产生 result；Tool Results 定义 result 的 envelope。
- [[harness/context/context-budgeting/index|Context Budgeting]] 决定 result 的哪些部分进入下一轮模型。
- [[harness/workspace/artifacts/index|Artifacts]] 是 result 中大型或持久产物的落点。

## 目录

- [[harness/tooling/tool-results/implementations/uxarts-agent|uxarts-agent 实现]]
