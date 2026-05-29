---
title: Agent Skills
type: concept
created: 2026-04-11
updated: 2026-04-12
sources:
  - llm/summaries/claude-agent-skills-docs
  - llm/summaries/how-we-use-skills
tags:
  - AI/Agent/Skills
---

# Agent Skills

Skills 是对 Claude 能力的模块化扩展——一个打包了指令、元数据和可选脚本的文件目录，按需加载，把通用 AI 变成领域专家。

---

## 核心设计洞察：Progressive Disclosure

Skills 的架构价值不在于"存储指令"，而在于**渐进式披露（Progressive Disclosure）**：

| 层级 | 加载时机 | Token 消耗 | 内容 |
|------|---------|-----------|------|
| 元数据 | 始终加载 | ~100 tokens/Skill | `name` + `description` |
| 指令 | Skill 被触发时 | 通常 <5k tokens | SKILL.md 主体 |
| 资源 | 按需访问 | 几乎无限制 | 脚本、参考文档、模板 |

只有元数据常驻上下文——这使得数十个 Skills 并存时，启动 token 消耗仍可控。这与 [[llm/concepts/agent-tool-design]] 中的 Progressive Disclosure 工具设计原则一脉相承。

---

## Skills 适合封装什么

官方文档描述了机制，生产实践（[[llm/summaries/how-we-use-skills]]）揭示了哪类知识值得打包为 Skill：

| 类型 | 核心价值 | 典型例子 |
|------|---------|---------|
| Library & API Reference | Claude 不知道内部库的正确用法 | `billing-lib`、`internal-cli` |
| Product Verification | 可重复的端到端验证流程 | `signup-flow-driver` |
| Data & Analysis | 连接数据栈，避免每次重新解释 | `funnel-query`、`grafana` |
| Business Automation | 把多步重复流程变成单条指令 | `standup-post`、`weekly-recap` |
| Coding Standards | 内部规范、架构约束 | `api-conventions`、`db-migration` |

**判断标准**：某个任务是否被重复触发？Claude 是否每次都需要相同的背景知识？如果是，就值得做成 Skill。

---

## 写好 Skill 的原则

官方规范 + 生产经验（Anthropic 内部数百个 Skill 在跑）综合出的核心原则：

**① 只写 Claude 不知道的信息**：通用知识不需要重复。Skill 是差量，不是全量。

**② 自由度匹配任务脆弱性**：

| 场景 | 做法 |
|------|------|
| 多路径都可行 | 文字指导原则，给 Claude 空间 |
| 有偏好但可变 | 伪代码或带参数模板 |
| 顺序严格/易出错 | 精确脚本 + "不要修改命令" |

**③ 引用深度最多一层**：SKILL.md → REFERENCE.md ✅，更深层 Claude 可能不读。

**④ description 决定触发时机**：`"[做什么]. Use when [触发场景]."` 写得越精准，误触发越少。

**⑤ 评估驱动开发**：先写测试场景再写 Skill，用最少指令让测试通过。

---

## 与 Prompt Caching 的关系

Skills 的元数据层（Level 1）是 cache prefix 的稳定组成部分，直接影响缓存命中率。Skills 数量增多时，**确保元数据顺序稳定**是维持高命中率的关键。见 [[llm/concepts/prompt-caching/system-design-principles]]。

---

## Open questions

- 当 Skill 数量增长到数百个时，元数据层的 token 消耗如何控制？
- 跨团队共享 Skill 时，版本管理和权限控制的最佳实践？

---

## Sources

- [[llm/summaries/claude-agent-skills-docs]] — 机制：Anthropic 官方 Agent Skills 文档
- [[llm/summaries/how-we-use-skills]] — 实践：Claude Code 团队生产经验
