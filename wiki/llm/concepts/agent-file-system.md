---
title: Agent File System
type: concept
created: 2026-04-11
updated: 2026-04-12
sources:
  - llm/summaries/files-are-all-you-need
  - llm/summaries/nodejs-vfs-pr
tags:
  - AI/Agent/Architecture
  - AI/Systems
---

# Agent File System

用文件系统抽象统一 AI Agent 的状态层——记忆、工具输出、通信、配置全部建模为文件，获得 Unix 管道的可组合性与可调试性。

---

## 为什么文件抽象有效

文件抽象的核心价值是**最小接口 + 统一语义**：不管底层是磁盘、内存、网络还是 FIFO，调用方只需 `open/read/write/close`。

这带来三个直接好处：
- **可组合**：Agent 之间像 Unix 管道连接，无需了解彼此内部实现
- **可调试**：推理步骤和中间结果持久化为文件，可直接 `cat` 检视
- **可恢复**：状态不在内存里，崩溃不丢，可从任意检查点重启

理论上看起来简洁（[[llm/summaries/files-are-all-you-need]]），工程上却代价不小（[[llm/summaries/nodejs-vfs-pr]]）。

---

## 理论与工程的落差

**理论层**把 Agent 的六类实体全部映射到文件：记忆、工具输出、配置、通信、状态、工作流。类比清晰，但停在概念层。

**工程层**揭示了真正的实现成本：

| 实现层级 | 内容 | 何时需要 |
|---------|------|---------|
| 最小可用 | 7 个核心原语 + read/write + FD 表 | 只需给 Agent 隔离工作空间 |
| 实用 | ReadStream/WriteStream + Overlay 模式 | 需要流式 I/O 或混合真实/虚拟路径 |
| 完整兼容 | 164+ 运行时拦截点 + hardlink + watcher | 需要透明替换现有代码的所有 fs 调用 |

从"最小可用"到"完整兼容"是巨大的工程鸿沟。给 Agent 用的文件系统通常只需要最小可用层。

---

## Overlay 模式：两个来源的交叉点

Overlay 在两个来源里都出现，但视角不同：

- **理论视角**：OverlayFS 对应"上下文注入"——给 Agent 注入额外知识而不修改基础能力
- **工程视角**：`vfs.create({ overlay: true })` 只拦截虚拟路径，其余 fall-through 到真实 fs

两者是同一机制的不同用途：前者用于扩展 Agent 的知识边界，后者用于渐进迁移和测试隔离。

---

## 实现要点

- **内存 VFS**：获得文件抽象的好处，同时避免磁盘 I/O。FD 从高位（如 10000）分配避免与 OS FD 命名空间冲突
- **Per-function hooks 优于 Proxy**：Proxy 有运行时开销且难以局部关闭；显式 hook 在 inactive 时成本为零
- **symlink 需要循环检测**，否则路径解析会无限递归

---

## Open questions

- 文件系统抽象是否足以处理多 Agent 并发写入时的状态一致性？
- 内存 VFS 在长时间运行 Agent 中的内存增长如何管理？

---

## Sources

- [[llm/summaries/files-are-all-you-need]] — 理论：Unix 哲学与 Agentic AI 系统设计
- [[llm/summaries/nodejs-vfs-pr]] — 工程：Node.js 内存文件系统 PR 分析
- [[llm/concepts/agent-tool-design]] — 文件系统作为工具输出的标准化接口
