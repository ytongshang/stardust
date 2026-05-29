---
title: Implementation Comparison
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - claude-code/src/services/compact/
  - claude-code/src/services/SessionMemory/
  - openclaw/src/agents/pi-embedded-runner/
  - openclaw/src/auto-reply/reply/
  - OpenHarness/src/openharness/services/compact/
  - OpenHarness/src/openharness/memory/
tags:
  - context-window
  - memory
  - architecture
  - comparison
---

# Implementation Comparison

> 三系统横向对比：Claude Code（TypeScript）、OpenClaw（TypeScript）、OpenHarness（Python）

---

## 全景对比表

| 维度 | Claude Code | OpenClaw | OpenHarness |
|------|------------|---------|-------------|
| **语言** | TypeScript | TypeScript | Python |
| **压缩层级数** | 2级（SM + LLM） | 2级（截断 + LLM） | 4级（微压缩→折叠→SM→LLM） |
| **记忆提取时机** | 每轮后异步（后置） | 压缩前强制（前置） | 手动/按需 |
| **跨会话存储** | Markdown 文件 | SQLite 数据库 | Markdown + JSON 快照 |
| **召回算法** | LLM 语义选择 | 向量+全文混合 | 关键词打分 |
| **Prompt 结构** | analysis + summary | 纯文本摘要 | analysis + summary |
| **摘要 Role** | user | user | user |
| **API 不变量保护** | 显式 adjust 函数 | sanitize 函数 | 验证后返回 None |
| **PTL 重试** | 按 round-trip 分组丢弃 | 按轮数截断 | 按轮数截断 |
| **可扩展性** | 配置化 | 插件化 Context Engine | 配置化 |

---

## 压缩流水线详解

### OpenHarness：四级渐进式（最透明）

```
Token 超标?
  ├─ Level 1 微压缩（清 tool results）────────→ 通常节省 20-40%
  │   ├─ 足够了? → 结束
  │   └─ 不够 ↓
  ├─ Level 2 上下文折叠（截断老文本）──────────→ 通常节省 10-20%
  │   ├─ 足够了? → 结束
  │   └─ 不够 ↓
  ├─ Level 3 Session Memory 确定性摘要 ─────────→ 无 LLM，确定性
  │   ├─ 可执行? → 结束
  │   └─ 消息太少 ↓
  └─ Level 4 LLM Full Compact ──────────────────→ 最强，耗时
```

**优势**：逻辑清晰，每一级都是独立模块，易于测试和复用。
**劣势**：没有利用已有的 Session Memory 跳过 Level 4，每次都可能触发 LLM 调用。

### Claude Code：Session Memory 优先（最高效）

```
Token 超标?
  ├─ 有 Session Memory? ──────────────────→ 直接用，无 LLM 调用
  │   ├─ 压缩后不超阈值? → 结束
  │   └─ 仍超 ↓
  └─ LLM Full Compact（最后手段）─────────→ 有 PTL 重试保护
```

**设计飞轮**：Session Memory 的持续异步提取使得多数情况下可以跳过 LLM 调用。
- 越早提取 Session Memory → 压缩越便宜
- 压缩越便宜 → 可以保持更长的有效对话

```
Session Memory 异步提取循环：
  每轮 query 结束
    → shouldExtractMemory()?
      → 是：forked agent 在后台更新 MEMORY.md
      → 否：跳过
  
  下次 compact 触发时：
    → 已有 Session Memory → 直接利用，无额外 LLM 调用
```

### OpenClaw：工具截断前置（最细粒度）

```
发送 API 请求前：
  ├─ 计算 tool results 总量
  ├─ 超过单条上限 (40K chars)? → 截断单条
  └─ 总量超 context 30%? → 按比例削减所有 tool results

Token 超标时：
  └─ LLM Full Compact（通过 pi-library）
```

**特色**：工具截断在**每次请求前**都执行，是预防性而非响应性的，token 使用更平稳。

---

## Compact Prompt 的异同

三个系统的摘要 prompt 高度相似，都来自同一设计哲学（OpenHarness 明显参考了 Claude Code 的设计）：

### 共同结构

```
[1] NO_TOOLS_PREAMBLE  — 禁止工具调用（防止模型"惯性"调用工具）
[2] <analysis> 指令    — 鼓励思考草稿，但草稿会被后处理剥离
[3] <summary> 9章结构  — 标准化输出格式
[4] NO_TOOLS_TRAILER   — 末尾再次重申禁止工具调用
```

### 关键差异

**Claude Code** 有三种 compact prompt 变体：
```
BASE_COMPACT_PROMPT          — 对全部历史做摘要
PARTIAL_COMPACT_PROMPT       — 只摘要 older 部分，newer 原样保留
PARTIAL_COMPACT_UP_TO_PROMPT — 摘要到第 N 轮，后续消息接在摘要后
```

增量 prompt 末尾追加提示：
```
Note: The most recent messages are being preserved verbatim and will follow
this summary. Focus your summary on the earlier portion of the conversation.
```

**OpenClaw** 通过 `customInstructions` 参数注入额外要求：
```typescript
await session.compact(params.customInstructions);
// customInstructions 可包含：项目特定知识、领域词汇表、格式偏好等
```

**OpenHarness** 类似，通过 `custom_instructions` 参数：
```python
compact_prompt = get_compact_prompt(custom_instructions)
# 如果有 custom_instructions，追加：
# "\n\nAdditional Instructions:\n{custom_instructions}"
```

---

## 记忆提取时机的哲学差异

这是三个系统最本质的设计差异：

### Claude Code：异步后置，最小干扰

```
main thread: [query 1] ──→ [query 2] ──→ [query 3]
                    ↓
                 post-hook
                    ↓
            background fork: [extract session memory]
```

- **优点**：不阻塞主对话，失败无影响
- **缺点**：如果进程意外退出，尚未提取的记忆丢失
- **保护机制**：`sequential()` 确保串行，`session_memory` querySource 防递归

### OpenClaw：同步前置，最安全

```
[query N] → [token超限] → [memory flush] → [compact] → [query N+1]
                              ↑
                      先持久化再压缩
```

- **优点**：压缩前记忆一定已落盘，数据安全性最高
- **缺点**：稍微增加压缩前的延迟
- **保护机制**：`compactionCount` 计数器 + context hash 去重，避免重复 flush

### OpenHarness：用户主动，最简单

```
用户请求 → 执行 add_memory_entry() → 写入文件 → 下轮 query 时自动召回
```

- **优点**：逻辑最简单，完全可预期
- **缺点**：需要用户或系统主动触发，依赖用户行为
- **适用**：自定义 Agent 场景，开发者完全掌控记忆管理时机

---

## 召回精度 vs 成本权衡

```
                    高精度
                      ↑
                      │   Claude Code
                      │   [LLM 语义选择]
                      │   · 理解语义意图
                      │   · 额外 API 调用成本
                      │
                      │           OpenClaw
                      │           [向量+BM25混合]
                      │           · 无 LLM 调用
                      │           · 需要向量索引基础设施
                      │
                      │                   OpenHarness
                      │                   [关键词打分]
                      │                   · 纯 CPU，零依赖
                      │                   · 依赖好的 description 字段
低成本 ←──────────────────────────────────────────────────→ 高成本
```

**关键洞察**：

Claude Code 用 LLM 做选择器的成本实际上很低——因为输入只有 header（文件名 + description），不是全文。一次选择调用约 500-1000 input tokens，远低于误召回不相关记忆所浪费的 tokens。

OpenHarness 的关键词打分看似简单，但通过对 frontmatter 字段加倍权重，有效地将"召回质量"的责任转移给了"记忆写入质量"。写好 description = 免费的召回优化。

OpenClaw 的向量搜索是三者中扩展性最好的，但需要 SQLite + 向量扩展的基础设施，在嵌入式或轻量场景下成本过高。

---

## 摘要消息的格式细节

三个系统都将摘要以 `role="user"` 插入，但格式有细微差异：

### Claude Code

```
This session is being continued from a previous conversation that has been compacted.

Summary:
[9 章结构化摘要内容]

If you need specific details that aren't captured in this summary, you can read
the full transcript at: /path/to/transcript.json

Recent messages are preserved verbatim below.
```

### OpenHarness

```
[9 章结构化摘要内容，已剥离 <analysis>，<summary> 标签替换为 "Summary:\n"]
```

更简洁，没有"你可以读全文"的引导，因为 OpenHarness 有会话快照文件（`latest.json`）可以从外部恢复。

### OpenClaw

```
[纯文本摘要，无特定结构要求]
```

因为 pi-library 控制摘要格式，并通过 `firstKeptEntryId` 精确标记了哪些消息是未摘要的"尾部"，恢复时直接从 transcript 读取，不依赖摘要的格式。

---

## Post-Compact 消息列表结构

压缩后，新的消息列表按以下顺序组装：

### Claude Code

```typescript
buildPostCompactMessages(result):
[
    boundaryMarker,     // SystemMessage: 标记压缩点，含 preCompactTokens/postCompactTokens 元数据
    ...summaryMessages, // UserMessage: 摘要内容
    ...messagesToKeep,  // 保留的原始消息（未压缩的尾部）
    ...attachments,     // 重注入的附件（如 plan 文件内容）
    ...hookResults,     // CLAUDE.md 等 session hooks 恢复的内容
]
```

`boundaryMarker` 是一个含元数据的系统消息，用于：
- 标记压缩发生的位置（便于 debug）
- 存储 pre/post token count（用于效果评估）
- `isCompactBoundaryMessage()` 检查 + 过滤

### OpenHarness

```python
build_post_compact_messages(result):
[
    *hook_results,      # pre-compact hooks 注入的消息（如项目说明）
    summary_msg,        # UserMessage: 摘要内容
    *messages_to_keep,  # 保留的原始消息
    *attachments,       # 附件（代码片段等）
]
```

### OpenClaw

通过 `session.buildSessionContext()` 动态重建，不在内存中保持完整列表：

```typescript
[
    { role: 'user', content: compactionEntry.summary },  // 压缩摘要
    ...keptMessages,  // firstKeptEntryId 之后的消息
]
```

特点：每次需要 context 时都从 transcript.jsonl 重建，保证与磁盘状态一致。

---

## 设计哲学总结

### Claude Code：以持续异步记忆为核心

Session Memory 不是附加功能，而是整个 context management 的**基础设施**。核心洞察：

> 与其每次压缩都调用 LLM，不如持续维护一份"滚动摘要"，压缩时直接使用。

这将 LLM Full Compact（昂贵）转化为 Session Memory Compact（几乎免费）。

### OpenClaw：可插拔架构，面向产品化

Context Engine 接口是 OpenClaw 最重要的设计决策。通过将 context 管理抽象为插件，OpenClaw 可以在不修改核心代码的情况下支持不同的记忆策略。

> 不要把一种记忆策略写死，而是定义清晰的接口，让不同场景用不同实现。

SQLite + 向量存储体现了这种面向未来的设计：今天用关键词，明天可以无缝升级到向量搜索。

### OpenHarness：明确渐进，低依赖，易理解

四级流水线是 OpenHarness 最大的优点：每一级都是独立的、无状态的函数，可以单独测试、单独使用。

> 如果你不知道该用哪种方案，就用最简单的、最容易理解的那种。

纯文件系统 + 关键词搜索，无向量数据库，无复杂异步逻辑，但对于大多数 Agent 系统已经足够。
