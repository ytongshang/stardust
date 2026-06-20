---
title: Concept Directory Template
type: template
created: 2026-06-20
updated: 2026-06-20
tags:
  - agent-harness
  - template
---

# Concept Directory Template

每个 harness 小概念建议是一个目录，而不是一个混合大文件。

## 目录结构

```text
concept-name/
  index.md
  implementations/
    uxarts-agent.md
    external-source-name.md
```

## index.md

写概念本身：

- 这个概念是什么。
- 它解决什么问题，为什么 agent harness 里需要单独建模。
- 常见怎么做：核心对象、流程、策略、边界和取舍。
- 这么做有什么优势，例如可维护性、可复用性、可测试性、可解释性、可靠性或成本控制。
- 它和相邻概念的边界是什么，避免把 application-specific implementation、运行时机制和上下文策略混在同一层级。
- 常见子问题。
- 本目录有哪些实现笔记。
- 什么样的外部材料值得记录。

不要只写一句定义。`index.md` 应该让读者明白这个概念为什么存在、通常怎么落地，以及什么样的外部实现值得继续研究。

不要在 `index.md` 里写太多具体项目实现。

## implementations/uxarts-agent.md

只写我们现在的实现：

- uxarts-agent / uxa-center 现在是什么。
- 做什么。
- 怎么做。
- 关键代码锚点。
- 不能丢的细节。

## implementations/external-source-name.md

写某个外部项目、文章、论文或 thread 的特别想法：

- Source。
- 属于哪个概念。
- 和 uxarts-agent 的对应关系。
- 新东西是什么。
- 实现细节。
- 是否值得后续借鉴。

如果没有新东西，只在目录已有笔记里简短记录，不要新增长文。
