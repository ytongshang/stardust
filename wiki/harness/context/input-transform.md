---
title: Input Transform
type: concept
created: 2026-06-20
updated: 2026-06-20
sources:
  - uxarts-agent/uxarts/web/modules/getui/data_processing/message_loader.py
  - uxarts-agent/uxarts/web/modules/getui/task/prototype/prototype_generate_user_input_creator.py
  - uxarts-agent/uxarts/web/modules/getui/task/uidraft/uidraft_user_input_createor.py
  - uxarts-agent/uxarts/web/modules/getui/task/memory/memory_wiki_user_input_creator.py
tags:
  - agent-harness
  - context
---

# Input Transform

## 这个概念是什么

Input Transform 是把产品原始输入转成 LLM 可消费 user message 的过程。它不是单纯的“用户文本拼接”，通常还要处理附件、引用、业务参数、图片、system-reminder 和上下文块。

通用 harness 层只需要知道：输入如何从 raw input 变成 prompt messages。业务层才知道 prototype、uidraft、wiki、sub-agent 各自是什么。

## uxarts-agent 现在做什么

现有不同 transform：

- prototype: `prototype_user_message_creator`
- uidraft create/edit/polish/page generation: `uidraft_*_user_message_creator`
- memory wiki: `memory_wiki_user_message_creator`
- sub-agent general-purpose: `general_purpose_user_message_creator`
- sub-agent read-only: `lightweight_user_message_creator`
- default: `default_user_input_creator`

## uxarts-agent 怎么做

通用入口在 `MessageLoader.as_run_item`：

```text
center item data
  -> parse UxUserInputItemData
  -> if process_status == 0 and raw_input exists:
       call user_input_message_create_func(raw_input, context)
     else:
       reuse raw_message
```

prototype transform 负责：

- 读取用户文本。
- 获取附件 message blocks。
- 保存 user documents。
- 根据 `business_type` 分发到 business handler。
- 插入 context reminder。

uidraft transform 负责：

- 处理 selection preview。
- 处理 UI draft 的图片/选区/编辑需求。
- 生成多模态 user message。

sub-agent transform 分两类：

- lightweight：只带任务文本和附件，不注入 heavy context。
- general-purpose：注入 system reminder、static context、per-turn context，接近主 agent。

## 不能丢的细节

- raw input transform 只能发生一次，历史轮次不能重复 transform。
- 附件既要作为 prompt content，也可能保存到 user documents。
- business handler 是业务扩展点，不应成为底层 harness 概念。
- read-only sub-agent 不应继承 main-like heavy context。

## Related Implementations

- 外部材料如果只是“把用户输入变成 message”，不算新。
- 值得记录的是更好的 input provenance、attachment budgeting、multi-modal prompt assembly 或 transform replay 设计。
