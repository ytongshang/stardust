---
title: "Prompt Caching — API Breakpoint Management"
type: concept
created: 2026-04-11
updated: 2026-04-11
sources:
  - https://platform.claude.com/docs/en/build-with-claude/prompt-caching
tags:
  - AI/LLM/Caching
---

# Prompt Caching — API Breakpoint Management

API 层面的 `cache_control` 断点管理策略，来自实际生产实现。

## 配额计算

```
可用配额 = 4
         - system/developer 消息已占用数
         - tools 参数已占用数          ← 容易遗漏
         - 保留的旧断点（1 个）
剩余配额 → 分配给 tool/user/assistant 各自最后一条消息
```

⚠️ 统计配额时必须包含 `tools` 参数中的 `cache_control` 数量。

## 多轮对话断点策略

正确做法：**同时保留旧断点 + 在最新消息上新增断点**，利用 20-block lookback 命中旧缓存。

```
Turn 1:  [a, b, c*]          → 写缓存于 c
Turn 2:  [a, b, c*, d*]      → d miss → lookback 命中 c → 仅处理 d，写缓存于 d
Turn 3:  [a, b, c*, d*, e*]  → e miss → lookback 命中 d → 仅处理 e
```

## 断点选择优先级

- 靠后（最新）的消息优先分配配额
- 在消息内选最后一个非空 text block 打标
- system/developer 消息的 `cache_control` 保持不变，不参与重新分配

## 反模式

> ❌ 每轮都清除所有旧 `cache_control` 再全部重新打标
> → lookback 失效，旧缓存无法命中

> ❌ 统计配额时只数 system/messages，忽略 tools 参数中的标记
> → 实际发出的断点总数超过 4，API 行为不可预期

> ❌ 查找 preserved 断点时不限定 text block，导致找到 image/tool_result block
> → 恢复失败、配额被无效占用

## 实现参考

`process_cache_control` 函数三阶段处理：

1. **预扫描**：找到第一个旧断点（preserved）、各角色最后一条消息、sys 消息占用配额
2. **配额计算**：确定 target 消息集合（最新 remaining_quota 条）
3. **构建结果**：sys 消息原样保留，target 消息新增断点，preserved 消息恢复旧断点，其余清除

关键细节：
- preserved 断点查找时限定 text block，与恢复逻辑条件一致（两侧对齐）
- preserved_msg_idx 从候选列表中排除，避免配额浪费
- 字符串 content 仅在需要操作时才转换为 list

## 相关页面

- [[llm/concepts/prompt-caching/index]] — 概览
- [[llm/concepts/prompt-caching/system-design-principles]] — 系统架构层
- [[llm/concepts/prompt-caching/compaction-cache-safe-forking]] — Compaction 场景
