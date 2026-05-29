---
title: Claude Prompting
type: concept
created: 2026-04-11
updated: 2026-04-12
sources:
  - raw/llm/articles/claude-prompting-best-practices
tags:
  - AI/LLM/Prompt
---
2
# Claude Prompting

针对 Claude 4.x（Opus 4.6、Sonnet 4.6、Haiku 4.5）的 prompt 工程参考，从通用原则到 Agentic 系统设计。

---

## 基本原则

**把 Claude 当作聪明但不了解你业务的新员工。** 黄金法则：把 prompt 给一个对任务一无所知的同事看，如果他会困惑，Claude 也会。

- **直接说清楚**：不要绕弯子，明确说明要做什么和为什么
- **示例优先**：控制输出格式最可靠的方式是 few-shot，建议 3-5 个，用 `<example>` 标签包裹
- **长上下文前置**：20k+ token 场景下，长文档放 prompt 顶部、query 在后，响应质量提升高达 30%

---

## Claude 4.x 关键变化

### 弃用 Prefill

| 旧用途 | 迁移方案 |
|--------|---------|
| 控制输出格式 | Structured Outputs |
| 去除开场白 | `"Respond directly without preamble"` |
| 续写中断内容 | 移到 user message 里说明 |

### Adaptive Thinking

取代旧版 `extended thinking` + `budget_tokens`，模型自主决定推理深度：

```python
client.messages.create(
    model="claude-opus-4-6",
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # low / medium / high / max
)
```

### 工具调用语气

旧版激进语言（`"CRITICAL: You MUST use this tool"`）在 Claude 4.x 上导致过度触发。改用普通描述语气。

---

## Agentic 系统设计要点

### 跨 context window 的长任务

1. 第一个 context window 专门搭建框架：写测试、创建初始化脚本
2. 用 JSON 文件维护任务列表，状态外化，不依赖模型记忆
3. 创建 `init.sh` 等初始化脚本，避免新 context 重复配置环境
4. 新 context 从读状态开始：先读 `progress.txt`、git logs，再继续工作

### 安全边界控制

在 prompt 中明确列举需要用户确认的操作：

```
在执行以下操作前必须暂停并确认：
- 删除文件或目录
- force push 到远程分支
- 发送消息到外部服务
```

### 防过度工程化

```xml
<minimize_overengineering>
只做被明确要求的最小改动。不添加未要求的错误处理、不重构周边代码、不增加注释。
</minimize_overengineering>
```

---

## 与 Prompt Caching 的关系

Prompt 结构设计直接影响缓存命中率：静态内容（系统 prompt、工具列表）前置构成稳定 cache prefix，动态内容（用户输入）后置。见 [[llm/concepts/prompt-caching/system-design-principles]]。

---

## Open questions

- Adaptive Thinking 的 effort 参数如何与成本/延迟做 tradeoff？
- 多轮对话中，长文档的 cache prefix 如何在 compaction 后保持稳定？

---

## Sources

- [[llm/summaries/claude-prompting-best-practices]] — Anthropic 官方 Claude 4.x Prompting 文档
- [[llm/concepts/prompt-caching/system-design-principles]] — Prompt 设计与缓存命中率
- [[llm/concepts/agent-tool-design]] — Agentic 场景下的工具调用设计
