---
title: "Claude Agent Skills — 官方文档笔记"
type: summary
created: 2026-03-28
updated: 2026-04-11
sources:
  - https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
  - https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart
  - https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
tags:
  - AI/Agent/Skills
origin_type: doc
author: "Anthropic"
---

# Claude Agent Skills — 官方文档笔记

## 📌 核心观点

> Agent Skills 是对 Claude 能力的模块化扩展，本质是一个文件目录：打包了指令、元数据和可选脚本，Claude 在需要时按需加载，把通用 AI 变成领域专家。

**Skills vs Prompt 的关键区别：**
- Prompt 是一次性对话级指令
- Skills 是持久化、可复用的能力包，在多个对话中自动加载、自动触发

---

## 📋 核心机制

### 三层加载架构（Progressive Disclosure）

| 层级 | 加载时机 | Token 消耗 | 内容 |
|------|---------|-----------|------|
| Level 1：元数据 | 启动时始终加载 | ~100 tokens/Skill | YAML frontmatter 中的 `name` + `description` |
| Level 2：指令 | Skill 被触发时 | 通常 <5k tokens | SKILL.md 主体内容 |
| Level 3：资源/代码 | 按需访问 | 几乎无限制 | 脚本、参考文档、模板等附属文件 |

**只有元数据常驻上下文**，其余内容 Claude 通过 bash 命令读取文件系统。

### 目录结构

```
my-skill/
├── SKILL.md          ← 必须，含 YAML frontmatter + 主体指令
├── REFERENCE.md      ← 详细参考文档（按需加载）
└── scripts/
    └── analyze.py    ← 工具脚本
```

### SKILL.md 最小 frontmatter

```yaml
---
name: pdf-processing
description: Extract text and tables from PDF files. Use when working with PDF files or when the user mentions PDFs or document extraction.
---
```

---

## 📋 编写最佳实践

**① 精简为王**：只写 Claude 不知道的信息，SKILL.md 主体控制在 500 行以内

**② 自由度匹配任务脆弱性**

| 场景 | 自由度 | 做法 |
|------|-------|------|
| 多路径都可行（如代码 review） | 高 | 给文字指导原则 |
| 有偏好但可变（如报告格式） | 中 | 给伪代码或带参数模板 |
| 顺序严格/易出错（如数据库迁移） | 低 | 给精确脚本 + "不要修改命令" |

**③ Description 写法**：`"[做什么]. Use when [触发场景]."`（第三人称）

**④ 引用深度最多一层**：SKILL.md → REFERENCE.md ✅，更深层 Claude 可能不读 ❌

**⑤ 设计反馈循环**：复杂任务要设计"执行→验证→修正"循环

**⑥ 评估驱动开发**：先写测试场景再写 Skill，最少指令让测试通过

### 各平台差异

| 平台 | 自定义 Skill | 网络访问 | 共享范围 |
|------|------------|---------|---------|
| Claude API | ✅（上传） | ❌ 无网络 | 整个 workspace |
| Claude Code | ✅（文件系统） | ✅ 完整 | 个人/项目 |
| Agent SDK | ✅（.claude/skills/） | 取决于配置 | - |

---

## 🔗 相关页面

- [[llm/summaries/how-we-use-skills]] — Claude Code 团队实战经验，与本文互补
- [[llm/summaries/prompt-caching-is-everything]] — Skills 的高效运行依赖 Prompt Caching
- [[llm/concepts/agent-skills]] — （待建）Skills 机制综合概念页
