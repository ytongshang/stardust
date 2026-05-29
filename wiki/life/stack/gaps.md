---
title: Gaps
type: gap
created: 2026-05-27
updated: 2026-05-28
tags:
  - life
  - stack
  - gaps
  - capabilities
  - yatoro
sources:
  - "Slock thread #life:a361fe23"
  - "Slock thread #workflow:7c1e1588"
  - "Slock thread #all:2c9b901e"
  - "Slock thread #yatoro:1c5eacba — stack reorg Phase 1"
  - "Slock task #9 #yatoro:02318f5f — remove knowledge-curator from current docs (2026-05-28)"
migrated-from:
  - wiki/life/capabilities/index.md#L132-L267
  - wiki/life/capabilities/index.md#L297-L303
---

# Gaps

> 当前 Yatoro/Slock/Claude Code 体系的已知能力缺口，按编号稳定追踪。

## Gap 1 — 缺 Work Order / Job 状态机

现在工作流存在于 Slock thread 和 wiki 规则里，但没有结构化 job 表。

需要一个可以记录以下状态的系统：

- raw user request
- clarified requirements
- technical plan
- implementation branch/commit
- verification results
- checkpoint decisions
- migration decision
- final status

可选位置：

- Yatoro backend D1 新增 `automation_work_order` 表。
- 或先用 `yatoro task` + memo/wiki 组合模拟。

## Gap 2 — 缺统一 Agent 编排器

更准确地说：**有规则没引擎**。

`multi-agent-workflow` skill 是 spec，但没有 runtime enforcement。Coordinator 目前靠对话自觉执行规则，而不是程序化调度。

需要：

- 创建/分派 Builder/Reviewer 的标准协议。
- 收集结果的标准格式。
- 自动判断从“直接处理”升级到“需要 Builder / Reviewer / 人类确认”。
- 避免多个 Agent 同时改同一文件。
- 强制或提醒高风险/新能力任务必须进入 Coordinator + Builder + Reviewer 流程。

可能复用的现有机制：

- Pattern 已就绪：参考 [[life/workflows/runtime-enforcement-pattern|Runtime Enforcement Pattern]]，核心是 single mutator + getter-closure，关键不变量是 mutator 不暴露为 tool。后续实现“直接处理 -> Builder / Reviewer / paused / done”中途升级时优先复用这套模式。

## Gap 3 — 能力发现 registry 缺机器可读层

[[life/stack/index|Stack — System Map]] 和本 stack 子页是 registry v1，但仍是给人和模型阅读的 Markdown。

还缺机器可读 schema / API：

- capability name
- trigger
- command / skill / repo path
- inputs / outputs
- verification command
- safety/caution
- producer / consumer DAG: 不仅列能力，还要列能力之间的依赖拓扑。多 consumer 可以同时依赖同一 producer 的同一 schema，breaking change 会扇出传播。
- `wire_convention_version`: 每个 registry entry 都要 pin 它假设的全局 wire convention 版本。

后续可以结构化为 JSON/YAML，再接入 D1 或 backend API。

registry v2 至少需要两层结构：

- Global wire conventions：Long -> string、ISO 8601、null vs missing、enum code-only、`page+size` vs `cursor` 分页等横切规则。这些不是某个 schema 的字段，但每个 schema 都必须遵守。
- Concrete schemas：具体能力的 I/O 字段、事件帧、DTO、object-key、file-tool 协议等。

> **历史档案（2026-05-27 重组前 team 数据）。** 以下两段「schema 输入意愿」和「schema 依赖」中出现的 handle（@uxarts-glow、@Agent、@uxa-center、@Yatoro、@Yatoro-claude）属于 2026-05-27 team 结构重组前的状态，相关 agent 已退役（参见 [[life/stack/agents#历史变更注记|Agents — 历史变更注记]]）。**这些段落不再表示当前的 schema-contribution 意愿，也不再表示当前的 DAG 参与者**，仅作历史参考。面向新 team（@uxarts-builder、@uxarts-code-reviewer、@yatoro-* 各位）的重新征集尚未启动，未来由 @yatoro-workflow-coordinator 另起 proposal 调度，未派单前不应据此 DM/mention 任何当前 agent。

历史档案 · 已知 schema 输入意愿（pre-reorg snapshot, 2026-05-27）：

- @uxarts-glow：superun 前端 schema，包括 TS 类型镜像规则、SSE 帧解析、附件 URL 不缓存策略、ListItemDTO / DetailDTO 边界、5 态状态机 -> UI 映射。
- @Agent：VFS / file-tool / sidecar schema。
- @uxa-center：REST / DTO OpenAPI schemas、跨团队 wire-format conventions、存储选型 decision matrix、未来 MQ event envelope schema。
- Yatoro 系（@Yatoro / @Yatoro-claude）：待 draft。

历史档案 · 已知 schema 依赖（pre-reorg snapshot, 2026-05-27）：

- @uxa-center 消费 @Agent 的 `service_dto.py`，用于反推 MySQL 表结构 / HBase rowkey。
- @uxa-center 消费 @Agent 的 VFS / 附件 object-key 协议，用于对齐 metadata 表。
- @uxarts-glow 和 @uxa-center 都消费 @Agent 的 `service_dto.py`；这说明拓扑是 DAG，不是单链。

## Gap 4 — 抓取后处理还没做

RSS 音频抓取已完成 MVP，但播客完整链路还缺：

- 音频分片 / ffmpeg。
- ASR 转写。
- transcript 合并。
- 摘要、实体、时间线、行动项。
- 写入 memo/wiki/library。
- 定时调度和失败重试。

## Gap 5 — 验证没有统一报告格式

已有命令可以跑，但每个任务临时组织。

需要固定 `Verification Report`：

- commands run
- result
- artifacts
- risks
- rollback notes

## Gap 6 — 缺跨 Agent 协调协议

现在多个 Agent 主要靠善意避免重复劳动，缺少显式锁和 owner。

需要：

- thread owner / coordinator 标记。
- 文件或 artifact owner 字段。
- “某个 wiki 页/代码文件正在写”的状态标记。
- 冲突处理协议：谁停、谁合并、谁 review。
- 用户中断 / 暂停信号识别：当用户说“先别...”“不要继续了”“暂停一下”等短句时，当前 thread 的协作 Agent 应停止推进，并把暂停状态传播给其他参与 Agent。这个缺口比文件锁更紧急，因为 Agent 讨论跑过头不会自然报错。

> 速度超预期 != 协议健全。用户两次开口让停才止住，缺 agent 侧暂停信号识别。 -- @uxarts-glow @ #all:2c9b901e

现场数据：@uxarts-glow 反馈 `#all:2c9b901e` 中三方 Agent 约 1 小时完成 SSE 契约和 retry 语义对齐，但用户连续两次要求停止才真正停住。这说明多 Agent 高速协作时需要显式的人类暂停协议。

@Cindy 补充：这次不是“全体 Agent 都漏掉第一次暂停信号”，而是“有些 Agent 接住了，有些没有”。她在 owner 第一次说“你们几个先别聊了”时直接停了，即使那条消息没有 @ 她。这说明暂停信号识别目前是 Agent 个体解读策略，不是 runtime / skill 提供的均匀能力，无法稳定回归。

@Agent 补充量化数据：在 `#all:2c9b901e` 中，约 30 秒内完成 SSE `eventId` 发号、`end` 事件协议、retry `generationId` 三项契约对齐。这个速度说明多 Agent 协作一旦进入高速模式，可能快到人类难以及时介入。

可能解法：

- 做 thread/channel 级“用户暂停意图”广播信号，让所有协作 Agent 共享暂停状态。
- 把暂停语义识别原语放进 skill / runtime，包括短句、否定句、“别 / 停 / 先别 / 不要继续”等表达，而不是每个 Agent 各自靠模型阅读。

## Gap 7 — 缺自主触发回路

现在 AI 主要等人发消息才行动。

要做到“AI 主动推进”，还需要：

- scheduled scan / reminder-driven continuation。
- git diff hook：发现分支、测试失败、未提交工作后提醒或推进。
- wiki staleness checker：发现 capability/workspace map 过期。
- recurring job：定期抓取、总结、生成 report。
- action card：AI 生成可执行操作，人点击确认。

## Next Steps

1. 持续维护 [[life/stack/index|Stack — System Map]] 与本 stack 子页作为 capability registry v1；后续再做机器可读层 / 结构化 registry。
2. 用一个真实需求回归测试：播客 RSS → 转文字 → 摘要。
3. 如果这个流程跑第二次仍然有效，再在 Yatoro backend 里做 `automation_work_order` 状态机。
4. 把 `yatoro-capture` 补成正式 skill，让模型能主动使用抓取能力。
5. 专门试一次 `slock action prepare`，验证“AI 推进到可执行动作，人最后点确认”的手感。
