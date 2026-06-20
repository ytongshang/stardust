---
title: Input Transform
type: concept
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - context
---

# Input Transform

Input Transform 是把产品原始输入转成 LLM 可消费 user message 的过程。它可以包含文本、附件、多模态内容、application params、引用、system reminder 和 context blocks。

## 它解决什么问题

用户输入在产品里通常不是一段纯文本，而是由表单字段、附件、引用对象、选中区域、图片、工作区状态和应用参数组成。Input Transform 负责把这些原始输入变成模型可以理解、可追踪、可预算的 user content。

它需要避免两个问题：

- 直接把产品对象塞进 prompt，导致来源、权限和上下文边界不清。
- 过早把输入拼成一大段文本，导致后续无法做 budgeting、replay 或 multimodal adapter。

## 常见做法

常见实现会先保留结构，再在最后一步渲染为 message：

- normalize：把文本、文件、图片、selection、references 转成统一 input parts。
- enrich：补充来源、权限、mime type、artifact id、workspace path 等 metadata。
- filter：按 permission、size、token budget、model capability 过滤或降级。
- render：按目标 model 的 message format 输出 text/image/file/tool-specific parts。
- record：保留 transformed input，方便 Event Log、replay 和 debug。

## 边界

- Input Transform 处理“用户给了什么”；[[harness/context/prompt-assembly/index|Prompt Assembly]] 处理“这些内容在最终 prompt 中怎么组织”。
- [[harness/context/context-injection/index|Context Injection]] 处理系统主动注入的 context，不应该和原始 user input 混成同一来源。
- [[harness/context/context-budgeting/index|Context Budgeting]] 可以影响 transform 的截断和降级，但 budgeting 本身不是 transform。

## 目录

- [[harness/context/input-transform/implementations/uxarts-agent|uxarts-agent 实现]]

## 后续收集

外部实现如果有新的 input provenance、attachment budgeting、多模态输入组织、transform replay 或 application input adapter 设计，可以在本目录下新增笔记。
