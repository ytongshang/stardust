---
title: Resources
type: resources
created: 2026-05-27
updated: 2026-05-27
tags:
  - life
  - stack
  - resources
  - yatoro
  - paths
sources:
  - "Slock thread #yatoro:1c5eacba msg=5c870466 — coordinator resource inventory 2026-05-27"
---

# Resources

> Yatoro / Slock / agent-kit / wiki / 本地工具状态的 canonical 物理路径。MEMORY 只存高频 fast path snapshot，详细以本页为准。

## Yatoro Skills

| 项 | 路径 / 命令 |
|---|---|
| skills source repo | `/Users/rancune/Work/rancune/agent-kit` |
| skills source dir | `/Users/rancune/Work/rancune/agent-kit/skills` |
| active skill source registry（唯一） | `/Users/rancune/.yatoro/cache/skills.json` |
| generated skill index（cache） | `/Users/rancune/.yatoro/cache/skills-index.json` |
| 命令面 | `yatoro skills source ...` / `yatoro skills index ...` / `yatoro skills install` / `yatoro skills doctor` / `yatoro skills fix` |

约定：`.claude` / `.codex` 等安装态目录不作为 canonical skill source。Skill source 只通过 `yatoro skills source list|add|remove` 维护。

## Yatoro CLI

| 项 | 路径 |
|---|---|
| CLI source | `/Users/rancune/Work/rancune/cloud/yatoro-monorepo/apps/cli` |
| active `yatoro` command（shim） | `/Users/rancune/Library/pnpm/yatoro` |
| CLI global pnpm link | `/Users/rancune/Library/pnpm/global/5/node_modules/@yatoro/cli` → monorepo `apps/cli` |
| CLI dist actually executed by shim | `/Users/rancune/Work/rancune/cloud/yatoro-monorepo/apps/cli/dist/index.js` |

验证：
- `command -v yatoro` 应为 pnpm shim。
- `pnpm list -g` 应显示 `@yatoro/cli@link` 指向 monorepo `apps/cli`。

## Monorepo

- root: `/Users/rancune/Work/rancune/cloud/yatoro-monorepo`
- apps: `apps/cli`（CLI 主体）、`apps/*`（其他）
- 通用命令: `pnpm -w typecheck` / `pnpm -w build` / `pnpm --filter <package> test`

详细子项目清单不在本页维护；进入 monorepo 后看 `pnpm-workspace.yaml` 与各 `package.json`。

## Stardust-Wiki / Obsidian Vault

| 项 | 路径 |
|---|---|
| vault root | `/Users/rancune/Library/Mobile Documents/iCloud~md~obsidian/Documents/Stardust-Wiki` |
| Obsidian config | `/Users/rancune/Library/Mobile Documents/iCloud~md~obsidian/Documents/Stardust-Wiki/.obsidian` |
| wiki content | `<vault>/wiki/{llm,life,...}` |
| 系统地图入口 | [[life/stack/index\|Stack — System Map]] |

特性：iCloud 同步，无 git；写入后等 iCloud sync 完成；多设备先让主写入设备完整同步再 reindex；批量改 wikilink 后用 Obsidian unresolved-link 面板验收。

## 本地 Yatoro State

| 路径 | 内容 | 注 |
|---|---|---|
| `/Users/rancune/.yatoro` | Yatoro CLI 本地状态根 | 不要删除任何子项 |
| `/Users/rancune/.yatoro/auth.json` | 凭据 | **敏感，不读** |
| `/Users/rancune/.yatoro/db/` | 本地数据库 | 不擅自改 |
| `/Users/rancune/.yatoro/capture-state.json` | capture 任务状态 | 备份后再清 |
| `/Users/rancune/.yatoro/cache/skills.json` | skill source registry（见 [[#Yatoro Skills]]） | canonical |
| `/Users/rancune/.yatoro/cache/skills-index.json` | skill 生成索引 | 可重建 |

## Maintenance

- 资源地图只在**结构性变化**时更新：路径迁移、新 workspace、CLI shim 切换、registry 重命名、新建敏感文件、目录重组等。
- 普通文件改动**不更新**本页，避免 wiki 变成 noise。
- 更新后同步 `updated:` frontmatter。
- MEMORY 中只放极少高频路径 fast-path snapshot，详细以本页为准；遇到不一致以本页 + `command -v` / `ls -la` 实测为准。
