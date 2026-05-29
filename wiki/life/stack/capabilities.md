---
title: Capabilities
type: index
created: 2026-05-27
updated: 2026-05-28
tags:
  - life
  - stack
  - capabilities
  - personal-automation
  - yatoro
sources:
  - "Slock thread #life:a361fe23"
  - "Yatoro CLI README and backend route scan on 2026-05-27"
  - "Slock thread #workflow:7c1e1588"
  - "Slock thread #yatoro:1c5eacba — stack reorg Phase 1"
  - "Slock task #8 #yatoro:aca60753 — memory system survey (2026-05-28)"
  - "Slock task #9 #yatoro:02318f5f — remove knowledge-curator from current docs (2026-05-28)"
migrated-from:
  - wiki/life/capabilities/index.md#L17-L19
  - wiki/life/capabilities/index.md#L21-L40
  - wiki/life/capabilities/index.md#L42-L58
  - wiki/life/capabilities/index.md#L97-L105
  - wiki/life/capabilities/index.md#L107-L115
  - wiki/life/capabilities/index.md#L124-L130
---

# Capabilities

> 当前 Yatoro/Slock/Claude Code 体系已经具备的能力地图。用于从“人用 AI 工具”升级到“AI 主动推进，人做关键选择”。

## 能力总览

| 层 | 已有能力 | 状态 |
|---|---|---|
| 入口 | Slock thread、task claim/update、Agent 协作、reminder/action card/attachment | 可用 |
| Agent 生态 | #yatoro 当前 3 个 agent：coordinator / builder / code-review；canonical 名单见 [[life/stack/agents\|Agents]] | 可用，默认 coordinator-first |
| 工作流 | Adaptive Flow：先用现有知识/能力直接处理；缺能力时再实现、review、沉淀 | 可用，见 [[life/workflows/request-to-automation\|Request to Automation]] |
| 工作区定位 | 高层 routing + Yatoro 物理路径分别见 [[life/stack/workspaces\|Workspaces]] / [[life/stack/resources\|Resources]] | 可用 |
| 项目执行 | Yatoro monorepo、Claude Code repo、skills repo | 可用 |
| 任务管理 | `yatoro task *`、提醒、附件、push cron | 可用 |
| 速记/知识 | `yatoro memo *`、Stardust-Wiki、llm-wiki skill | 可用 |
| 文件/媒体 | `yatoro lib *` 上传/导入/R2/CDN、图库 folder/asset | 可用 |
| 抓取 | `yatoro capture rss`，RSS 音频抓取到 CDN | MVP 可用 |
| AI 生成 | `yatoro image gen/edit`，结果入图库 | 可用 |
| ASR/Voice | `/asr` 讯飞签名、`/voice` transcript CRUD | 有基础能力，未接入播客批处理 |
| Backend 平台 | Cloudflare Workers + D1 + R2 + Cron + OpenAPI | 可用 |
| IM / Channel bridge | Claude Code repo 中有 weixin / dingtalk / feishu channel + MCP server 实现 | 可用性待按环境验证 |
| MCP / Integrations | Claude Code channel MCP；Slock integration command 存在但当前注册服务为空 | 部分可用 |
| 验证 | typecheck/build/dry-run/e2e 手动执行 | 可用但未统一编排 |
| 长期沉淀 | MEMORY、Obsidian wiki、skills、automation / repo docs；memory 设计见 [[life/stack/memory\|Memory]] | 可用 |

## 当前已能自动推进的事

### 需求到工作流

- 根据用户自然语言需求判断：直接回答、直接执行、查现有资产、补 wiki/docs、补 skill、补 MEMORY、补 automation/code，或需要 Builder/Reviewer。
- 生成任务卡、技术方案、实现计划、验证结果、迁移决策。
- 在 Slock thread 中持续汇报。
- 小任务可由一个 Agent 直接完成。
- 当出现新能力或非平凡实现时引入 Builder；当有实现风险、命令语义、安全边界、测试缺口或需要第二意见时引入 Reviewer。

### Slock 原生推进能力

- `slock task create/claim/update`：把工作显式化，避免重复劳动。
- `slock reminder schedule/snooze/update`：让 Agent 在未来时间点继续推进，而不是依赖人再次提醒。
- `slock attachment upload/view`：在 Slock 中流转文件、截图、报告。
- `slock action prepare`：准备 action card，让 AI 先推进到可执行方案，人最后点击确认；这是“AI 主动推进，人最后选择”的关键原语。
- `slock integration login`：为第三方服务准备 Agent 登录；当前 `slock integration list` 显示未注册服务，后续接入后可用。

## IM / Channel bridge

Claude Code repo 已有外部消息源接入实现：

- Weixin：`channels/weixin`，MCP server + `/weixin:login/status/logout`。
- DingTalk：`channels/dingtalk`，MCP server + `/dingtalk:access ...`。
- Feishu/Lark：`channels/feishu`，MCP server + `/feishu:access ...`。

这些能力适合做“日常入口”和外部通知，但需要按具体环境确认登录态、凭据和 MCP 连接状态。

## 执行个人系统操作

通过 `yatoro` CLI：

- `task`：任务、清单、标签、提醒、附件。
- `memo`：速记、标签、搜索、反链、统计。
- `lib`：上传文件到 R2/CDN、导入外链、管理图库资产。
- `image`：AI 生成/编辑图片并入图库。
- `capture`：抓取外部 source 文件；当前支持 podcast RSS audio enclosure。

「写代码和验证」部分已迁入 [[life/stack/workspaces|Workspaces]]（按 workspace 分别列 verification commands）。

## 沉淀知识

- 项目命令/模式 → repo docs。
- 概念/方法论 → `wiki/llm`。
- 日常 SOP / 工作流 → `wiki/life`（包含本 stack/）。
- Agent 自身上下文 → `MEMORY.md` / notes。
- 模型执行规则 → skills。
- Memory 分层与沉淀策略 → [[life/stack/memory|Memory]]。
