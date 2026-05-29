---
title: Case Studies
type: case
created: 2026-05-27
updated: 2026-05-28
tags:
  - life
  - stack
  - case-studies
  - violations
  - retrospective
sources:
  - "Slock thread #life:a361fe23"
  - "Slock thread #workflow:7c1e1588"
  - "Slock thread #yatoro:1c5eacba — stack reorg Phase 1"
  - "Slock task #9 #yatoro:02318f5f — keep current workflow docs latest-only (2026-05-28)"
migrated-from:
  - wiki/life/capabilities/index.md#L269-L295
---

# Case Studies

> 已知工作流违规与复盘记录。新案例追加到末尾，不删除旧记录。

## Case 1 — capability map 本应升级复核，但实际单 Agent 完成

本系统初版能力盘点跨越 Slock、Yatoro CLI、backend、skills、wiki、workspace map，按当前 [[life/workflows/request-to-automation|Request to Automation]] 应属于“需要 Reviewer 或至少第二意见”的任务。

实际发生：

- Coordinator role: 未显式指定。
- Builder role: 单 Agent 直接完成。
- Reviewer role: @Yatoro-claude 事后补 review。
- Checkpoint: 跳过。

结论：这不是失败，是 runtime 数据。说明升级复核不会自然发生；需要编排器或至少需要 skill/runtime 在识别高风险/大范围任务时提醒进入 Coordinator + Builder + Reviewer。

## Case 2 — workflow 设计讨论未先查已有资产

2026-05-27 在 `#workflow:7c1e1588` 中，用户提出“把生活/工作需求变成 AI 驱动工作流，多 Agent 协作设计和自动化”的方向。

实际发生：

- Coordinator role: 一开始未显式指定；多个 Agent 同时补充框架。
- Asset lookup: 未先查 `agent-kit/skills/multi-agent-workflow`、[[life/workflows/request-to-automation|Request to Automation]]、本系统能力地图、workspace map。
- 结果：重复设计了已有的分诊、Coordinator/Builder/Reviewer、自动化/skill/wiki/memory 迁移决策等内容。
- 修正：用户随后指定 Yatoro-codex（稳定 handle 为 `@Yatoro`，display name 可能调整）担任 Coordinator，由 Coordinator 开始把这次反例沉淀到 wiki。

结论：涉及 workflow、自动化、skill、wiki、memory 的需求，第一步必须是资产检索和分诊，不能默认开新设计。复杂任务必须先指定 Coordinator；未指定时，由第一个接手的 Yatoro 系 Agent 临时担任，并必须在接手后的第一条 thread 回复里显式声明。这个案例同样说明规则不会自然执行，需要 [[life/workflows/runtime-enforcement-pattern|Runtime Enforcement Pattern]] 一类机制把提醒和 gate 前置。
