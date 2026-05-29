---
title: Agents
type: agents
created: 2026-05-27
updated: 2026-05-28
tags:
  - life
  - stack
  - agents
  - roster
  - yatoro
sources:
  - "Slock thread #yatoro:1c5eacba — stack reorg Phase 1"
  - "Slock thread #all and #yatoro 2026-05-27 — team reorg from runtime-based to role-based"
  - "slock server info verified 2026-05-27"
  - "slock channel members #yatoro verified 2026-05-28"
  - "Slock task #9 #yatoro:02318f5f — remove knowledge-curator from current docs"
migrated-from:
  - wiki/life/capabilities/index.md#L60-L86
---

# Agents

当前 #yatoro channel 的工作组。**2026-05-28 起，本群不再保留独立 knowledge-curator。**知识整理、wiki、skills、MEMORY 清理是 workflow coordinator 的能力之一；只有需要新能力实现或风险复核时，才拉 builder / code-review。

## Yatoro 方向

当前 #yatoro 成员（`slock channel members #yatoro` 2026-05-28 验证）：

| Handle | 角色定位 |
|---|---|
| `@yatoro-workflow-coordinator` | 默认入口；负责回答、查现有资产、轻量执行、scope、分诊、沉淀决策；也负责 wiki/skill/MEMORY 结构清理 |
| `@yatoro-builder` | 新能力或非平凡实现；包括 Yatoro 代码、CLI、脚本、agent-kit skills |
| `@yatoro-code-review` | 独立复核；用于实现风险、命令语义、安全边界、测试缺口、需要第二意见的方案 |

## Handle 与 display name 约定

**@-mention 用稳定 handle，不是 display name。** display name 可能在 thread 里临时漂移（历史例：@Yatoro 曾短暂以 "Yatoro-codex" 出现），`slock server info` 是 canonical source。

## 历史变更注记

2026-05-27 重组前 → 2026-05-28 最新状态：原 @Yatoro / @Yatoro-claude / @Agent / @uxarts-glow / @uxa-center / @CodeReview 等旧 handle 不再作为当前 #yatoro 工作组依据；2026-05-27 曾短暂存在 4 个 yatoro-* 角色，2026-05-28 起收敛为 3 个：coordinator / builder / code-review。[[life/stack/gaps|Gaps]] 里的 Gap #3 schema 贡献意愿、schema 依赖 DAG 若引用旧 handle，只作历史参考，不表示当前参与者。

## Maintenance

本页由 #yatoro 当前 coordinator 维护。server 里可能存在其他 agent（如 uxarts 或 onboarding 方向），但不默认进入 #yatoro 流程；需要跨 channel 协作时，由用户或 coordinator 显式邀请。
