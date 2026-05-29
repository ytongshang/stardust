---
title: "Lessons from Building Claude Code: How We Use Skills"
type: summary
created: 2026-03-26
updated: 2026-04-11
sources:
  - https://www.anthropic.com/engineering/claude-code-skills
tags:
  - AI/Agent/Skills
origin_type: blog
author: "Anthropic"
rating: ⭐⭐⭐⭐⭐
---

# Lessons from Building Claude Code: How We Use Skills

## 📌 核心观点

> Anthropic 内部大规模使用 Skills（数百个在跑），总结了哪类 Skill 值得做、怎么写好 Skill、如何分发的系统性经验。核心洞察：**Skill 是文件夹不是文件**，整个目录结构都是上下文工程的一部分。

---

## 📋 9 类 Skill 分类

| 类型 | 描述 | 典型例子 |
|------|------|---------|
| Library & API Reference | 内部库/CLI/SDK 的正确用法 | `billing-lib`、`internal-platform-cli` |
| Product Verification | 验证代码正确性，配 Playwright/tmux | `signup-flow-driver`、`checkout-verifier` |
| Data & Analysis | 连接数据和监控栈 | `funnel-query`、`grafana` |
| Business Automation | 把重复流程打包成单条指令 | `standup-post`、`weekly-recap` |
| Scaffolding & Templates | 生成特定框架的样板代码 | `new-migration`、`create-app` |
| Code Quality & Review | 代码审查和质量执行 | `adversarial-review`、`code-style` |
| CI/CD & Deployment | 拉取/推送/部署代码 | `babysit-pr`、`deploy-<service>` |
| Incident Runbooks | 症状 → 多工具调查 → 结构化报告 | `oncall-runner`、`log-correlator` |
| Infrastructure Ops | 日常运维，带安全护栏 | `<resource>-orphans`、`cost-investigation` |

---

## 📋 编写核心技巧

**1. Skip the Obvious** — 只写能把 Claude 推出常规思维的信息

**2. Build a Gotchas Section（最高价值区）** — 从实际失败点积累，持续更新

**3. Progressive Disclosure** — Skill 是文件夹。把详细内容拆成子文件（`references/api.md`），按需读取

**4. Don't Railroad Claude** — 给足信息，保留灵活性。用意图描述替代逐步骤处方：
- ❌ Step 1: Run git log... Step 2: Run git cherry-pick...
- ✅ Cherry-pick the commit onto a clean branch. Resolve conflicts preserving intent.

**5. config.json 模式** — 用 `!cat ${CLAUDE_SKILL_DIR}/config.json` 缓存用户首次配置

**6. Description = Trigger** — Claude 启动时扫描 description 来决定用哪个 Skill，description 是触发条件不是功能摘要

**7. On-Demand Hooks** — Skill 可注册只在该 Skill 激活期间生效的 Hook（如 `/careful` 阻止危险操作）

---

## 💡 关键洞察

- **Gotchas 是 Skill 中信号最强的内容**：把失败点结构化记录，随时间积累
- **渐进式披露控制 token 消耗**：只在需要时加载详细内容
- **Agent 上下文工程 = 文件系统结构**：文件布局本身就是上下文的一部分，不只是 prompt

---

## 🔗 相关页面

- [[llm/summaries/claude-agent-skills-docs]] — 官方文档，与本文互补
- [[llm/summaries/prompt-caching-is-everything]] — Skills 与 Caching 组合是高效运行底座
- [[llm/summaries/seeing-like-an-agent]] — 工具设计哲学（同系列）

## 📎 原文摘录

> "a skill is a folder, not just a markdown file. You should think of the entire file system as a form of context engineering and progressive disclosure."

> "The highest-signal content in any skill is the Gotchas section."
