---
title: "Claude Prompting Best Practices"
type: summary
created: 2026-03-27
updated: 2026-04-11
sources:
  - https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
tags:
  - AI/LLM/Prompt
origin_type: doc
author: "Anthropic"
---

# Claude Prompting Best Practices

## 📌 核心观点

> 针对 Claude 最新模型（Opus 4.6、Sonnet 4.6、Haiku 4.5）的 prompt 工程全面参考指南，涵盖通用原则、输出控制、工具调用、思考推理和 Agentic 系统五大维度。

---

## 📋 通用原则

**清晰直接**：把 Claude 当作聪明但不了解你业务的新员工。"黄金法则"：把 prompt 给一个对任务一无所知的同事看，如果他会困惑，Claude 也会。

**添加背景上下文**：说明指令背后的原因，有助于 Claude 泛化。

**使用示例（Few-shot prompting）**：示例是控制输出格式最可靠的方式，建议 3-5 个，用 `<example>` 标签包裹。

**长上下文（20k+ tokens）**：把长文档放在 prompt 顶部，query 在后，可提升高达 30% 响应质量。

---

## 📋 Claude 4.x 关键变化

**弃用 Prefill**：迁移方案：
- 控制输出格式 → Structured Outputs
- 去除开场白 → `"Respond directly without preamble"`
- 续写中断 → 移到 user message 说明

**Adaptive Thinking（新默认模式）**：
```python
# 新版（取代 extended thinking + budget_tokens）
client.messages.create(
    model="claude-opus-4-6",
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # low / medium / high / max
    ...
)
```

**工具调用语气**：旧版激进语言（`"CRITICAL: You MUST use this tool"`）现在会导致过度触发，改用普通语气。

---

## 📋 Agentic 系统要点

**多 context window 工作流最佳实践**：
1. 第一个 context window 专门搭建框架（写测试、创建脚本）
2. 让 Claude 在 JSON 中维护测试列表
3. 创建 `init.sh` 等初始化脚本，避免重复工作
4. 新 context 从空白开始时，先读 `progress.txt`、git logs

**自主性安全控制**：在 prompt 中列举需要用户确认的操作（删文件、force push、发消息等）

**防过度工程化**：`<minimize_overengineering>` 风格 prompt，只做被要求的最小改动

---

## 🔗 相关页面

- [[llm/concepts/prompt-caching/system-design-principles]] — Prompt 设计与缓存命中率的关系
- [[llm/concepts/claude-prompting]] — （待建）综合概念页

## 📎 原文摘录

> "Think of Claude as a brilliant but new employee who lacks context on your norms and workflows."
