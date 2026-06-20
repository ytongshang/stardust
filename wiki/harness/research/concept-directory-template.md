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
  uxarts-agent.md
  external-source-name.md
```

## index.md

写概念本身：

- 这个概念是什么。
- 它解决什么问题。
- 常见子问题。
- 本目录有哪些实现笔记。
- 什么样的外部材料值得记录。

不要在 `index.md` 里写太多具体项目实现。

## uxarts-agent.md

只写我们现在的实现：

- uxarts-agent / uxa-center 现在是什么。
- 做什么。
- 怎么做。
- 关键代码锚点。
- 不能丢的细节。

## external-source-name.md

写某个外部项目、文章、论文或 thread 的特别想法：

- Source。
- 属于哪个概念。
- 和 uxarts-agent 的对应关系。
- 新东西是什么。
- 实现细节。
- 是否值得后续借鉴。

如果没有新东西，只在目录已有笔记里简短记录，不要新增长文。
