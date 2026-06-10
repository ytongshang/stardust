---
title: Stack — Yatoro System Map
type: index
created: 2026-05-27
updated: 2026-05-30
tags:
  - life
  - stack
  - yatoro
  - personal-automation
sources:
  - "Slock thread #yatoro:1c5eacba — stack reorg Phase 1 (2026-05-27)"
  - "Slock task #8 #yatoro:aca60753 — memory system survey (2026-05-28)"
  - "Slock task #9 #yatoro:02318f5f — remove knowledge-curator from current docs (2026-05-28)"
  - "Slock task #11 #yatoro:60f8bbf7 — memory pipeline executable design (2026-05-30)"
  - "Slock thread #life:a361fe23"
  - "Slock thread #workflow:7c1e1588"
---

# Stack — Yatoro 系统地图

> 我们这套 Yatoro/Slock/agent-kit 系统的查询入口。从这里定位，再进具体子页。

## 入口分流

| 我想知道... | 去哪 |
|---|---|
| 任务进哪个 workspace？目录路由是什么？怎么验证？ | [[life/stack/workspaces\|Workspaces]] |
| skills / CLI / wiki / monorepo 的物理路径在哪？CLI shim 指哪？ | [[life/stack/resources\|Resources]] |
| 我们这套系统能做什么？已有 capability 有哪些？ | [[life/stack/capabilities\|Capabilities]] |
| 当前 Agent 团队有哪些角色？谁负责什么？ | [[life/stack/agents\|Agents]] |
| Memory 怎么分层、开源实现怎么做？ | [[life/stack/memory\|Memory]] |
| Yatoro 怎么增量发现会话、抽候选、晋升 wiki/skill/automation？ | [[life/stack/memory-pipeline\|Memory Pipeline]] |
| 当前已知的能力缺口和待办？ | [[life/stack/gaps\|Gaps]] |
| 历史违规复盘和经验？ | [[life/stack/case-studies\|Case Studies]] |
| 完整工作流 SOP 是什么？ | [[life/workflows/request-to-automation\|Request to Automation]] |
| `runtime gate` / single mutator 模式？ | [[life/workflows/runtime-enforcement-pattern\|Runtime Enforcement Pattern]] |

## How to navigate

- 先查 [[life/stack/workspaces|Workspaces]]，定位代码、wiki、skills、资料的入口目录。
- 用 `yatoro skills index list` 或 `yatoro skills index search <query>` 查当前可用 skill。Skill source 只以 `~/.yatoro/cache/skills.json` 为准，通过 `yatoro skills source list|add|remove` 管理；`.claude`、`.codex` 等安装态目录不作为 canonical source。
- Claude Code channel/MCP implementations: `channels/weixin`、`channels/dingtalk`、`channels/feishu`。

## Frontmatter 规约（本 stack 所有页面遵循）

```yaml
title: <human-readable，可含中文>
type: index | reference | log | gap | case | agents | resources | workspaces | memory
created: <继承源文件 created；纯新页用执行日>
updated: <最近修改日，YYYY-MM-DD>
tags:
  - life
  - stack
  - <topic>
  - <继承源 tag>
sources:
  - "Slock thread #yatoro:1c5eacba — stack reorg Phase 1"
  - <继承源 sources>
migrated-from:                              # 仅迁移页有；list 形式
  - <旧路径#段落锚或行段，如 wiki/life/capabilities/index.md#L60-L86>
owner: <agent-handle>                       # 可选；标注页面 maintenance owner
cross-edit-protocol: <协议简述>             # 可选；与 owner 配套
```

- `migrated-from` 用 list 形式，允许一页合并多个源段落；`grep -r "migrated-from:" wiki/life/stack/` 可机器审计迁移完整性。
- `owner` / `cross-edit-protocol` 默认不写；仅在有特定维护方时启用。
