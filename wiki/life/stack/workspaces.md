---
title: Workspaces
type: workspaces
created: 2026-05-27
updated: 2026-05-27
tags:
  - life
  - stack
  - workspaces
  - routing
  - personal-automation
sources:
  - "Slock thread #life:a361fe23"
  - "Slock thread #yatoro:1c5eacba — stack reorg Phase 1"
migrated-from:
  - wiki/life/workspaces/index.md#L1-L78
  - wiki/life/capabilities/index.md#L117-L122
---

# Workspaces

> Agent 接到任务后先查这里，确认应该进入哪一类工作区，并知道该如何在该工作区里跑验证。

## 原则

- 先确认 workspace 和现有能力，再开始实现。
- 本页只纳管主要入口目录：`~/work`、iCloud Obsidian vault、全局工具状态目录。
- 未列出的目录默认不主动扫描、不总结、不纳管；需要时由用户明确给出路径或授权。
- 现在先不拆成多个文件。等某一类目录稳定需要细规则时，再拆 `work.md` / `icloud.md` / `tool-state.md`。

## Quick Routing

| 任务类型 | 先看哪里 |
|---|---|
| 代码仓库、Yatoro、Claude Code repo、公司/开源项目、脚本 | [[#Work Root]] |
| Obsidian wiki、VK、长期知识、Life/LLM SOP | [[#iCloud Obsidian]] |
| Claude/Codex/Slock/Yatoro CLI 配置、sessions、agent workspace、本地状态 | [[#Global Tool State]] |

## Work Root

- **Path**: `/Users/rancune/work`
- **Purpose**: 本机主要代码和项目工作区入口。
- **Use when**:
  - 用户提到代码、repo、monorepo、CLI、backend、skills、channel scripts、公司项目、开源项目。
  - 需要先找现有实现、命令、README、package 配置、git 状态。
- **How to use**:
  - 先在此根目录下按任务关键词定位具体 repo。
  - 进入具体 repo 后再读 README / docs / package 配置 / git status。
  - 不在本页展开子项目清单，避免 workspace map 变成项目索引。
- **Verification commands**（常用，按 repo 进入后再选）:
  - `git status`、`git log --oneline -10` 确认分支与最近改动。
  - Yatoro monorepo: `pnpm -w typecheck`、`pnpm -w build`、`pnpm --filter @yatoro/cli test`、`yatoro skills doctor`。
  - 通用 Node 项目: `pnpm install`、`pnpm test`、`pnpm build`。
  - dry-run/end-to-end：按 repo 自带 README 给出的命令优先。
- **Caution**:
  - 进入任何 repo 前先检查 git 状态。
  - 不要改动与当前任务无关的 repo 或文件。

## iCloud Obsidian

- **Path**: `/Users/rancune/Library/Mobile Documents/iCloud~md~obsidian/Documents/Stardust-Wiki`
- **Purpose**: iCloud Obsidian vault / personal VK / long-term knowledge base。
- **Use when**:
  - 需要沉淀概念、SOP、工作流、能力地图、workspace map。
  - 需要更新 LLM/Agent 方法论或 Life/practice 工作流。
- **Important areas**:
  - `wiki/llm`：LLM / AI / Agent 概念和方法论。
  - `wiki/life`：日常生活、个人工作流、自动化 SOP，含本 `stack/` 系统地图。
- **Verification commands**:
  - 写入后等 iCloud sync 完成；多设备场景需主写入设备先完整同步再让其他设备 reindex。
  - Obsidian unresolved-link 面板：批量改 wikilink 后用来确认无坏链。
  - vault 无 git，回滚靠手动备份；批量改前考虑先 zip 备份关键页。
- **Caution**:
  - 遵守 vault 的 `CLAUDE.md` 和 frontmatter 规范。
  - 概念/方法论放 `wiki/llm`；具体日常 SOP 和实践放 `wiki/life`。

## Global Tool State

这些目录不是普通项目，是工具本地状态。默认只读；只有用户明确要求修配置、迁移、排障时才修改。

| Path | Purpose | Caution |
|---|---|---|
| `/Users/rancune/.claude` | Claude Code 配置、plugins、skills cache、sessions、history | 不要删除 `settings.json`、`auth.json`、`sessions/` |
| `/Users/rancune/.codex` | OpenAI Codex CLI 配置、skills、plugins cache | 默认只读 |
| `/Users/rancune/.slock` | Slock daemon 状态与各 agent workspace | 不要删除 `agent-proxy-tokens/`、agent UUID 目录 |
| `/Users/rancune/.yatoro` | Yatoro CLI 本地状态（cache、auth、db） | 不要删除 `auth.json`、`db/`、`capture-state.json`；详细物理路径见 [[life/stack/resources\|Resources]] |

- **Verification commands**:
  - `yatoro skills doctor` 检查 skill source / index 一致性。
  - `yatoro skills source list` 看 source registry。
  - `command -v yatoro` 确认 CLI shim 路径。
  - `ls -la <path>` 看 agent workspace、cache、auth 文件存在性，但不读 `auth.json` 内容。

## 写代码和验证（通用能力）

迁自旧 capabilities：

- 能在 monorepo 中读代码、切分支、提交、实现。
- 能跑 CLI typecheck/build。
- 能真实 dry-run 或端到端验证，例如已验证 RSS 音频抓取到 CDN。
- 能用 `uv run --with pyyaml /Users/rancune/.codex/skills/.system/skill-creator/scripts/quick_validate.py <skill-dir>` 验证 skill frontmatter/结构。

## Maintenance

- 新增主入口目录时，先确认它是不是长期入口，而不是一次性项目。
- 项目级细节优先写入项目自己的 README/docs，或写到更具体的 wiki 页面。
- 如果某个入口目录开始需要大量规则，再从本页拆出独立文件并在 Quick Routing 中链接。
- Yatoro/skills/CLI/monorepo 等物理路径细节单独放 [[life/stack/resources|Resources]]，本页只做高层 routing。
