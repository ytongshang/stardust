---
title: Request to Automation
type: workflow
created: 2026-05-27
updated: 2026-05-28
sources:
  - "Slock thread #life:a361fe23 — 多 Agent 工作流设计讨论 (2026-05-27)"
  - "Slock thread #workflow:7c1e1588 — workflow triage / existing-asset lookup correction (2026-05-27)"
  - "Slock task #8 #yatoro:aca60753 — memory system survey (2026-05-28)"
  - "Slock task #9 #yatoro:02318f5f — remove knowledge-curator from current docs (2026-05-28)"
tags:
  - life
  - workflow
  - personal-automation
  - slock
---

# Request to Automation

> 方法论背景：[[llm/concepts/multi-agent-workflow/index|Multi-Agent Workflow]]

当我在 Slock 里提出问题或需求时，用这个 SOP 选择最轻的有效动作：能直接回答就直接回答；现有能力能做就直接执行；能力缺失时再补 docs、skills、MEMORY、automation 或代码。

这是一个持续进化流程，不是固定流水线。

## 默认流程

1. **理解请求**：明确用户真正要的是回答、执行、方案、调研、文档、实现，还是能力升级。
2. **查现有资产**：按需查 [[life/stack/index|Stack — Yatoro System Map]]、skills index、repo docs、wiki、MEMORY、代码。
3. **直接处理**：如果现有知识/能力足够，coordinator 直接回答或执行，并汇报验证结果。
4. **发现缺口**：如果现有能力不够，判断要补什么：docs/wiki、skill、MEMORY、automation、repo code、workflow。
5. **按需升级**：只有出现新能力或非平凡实现时拉 @yatoro-builder；只有实现风险、命令语义、安全边界、测试缺口或需要第二意见时拉 @yatoro-code-review。
6. **验证**：跑必要的 typecheck、测试、dry-run、doctor、smoke test 或真实端到端。
7. **沉淀决策**：决定产物是否进入 repo docs、Obsidian wiki、skill、MEMORY、automation script，或只留在 thread。
8. **清理旧信息**：如果新结论替代旧规则，更新 canonical 文档，避免 MEMORY/wiki/skill 同时漂移。

## 资产检索

涉及 workflow、自动化、skill、wiki、memory 的需求，不能直接重新设计。Coordinator 先做最小检索：

1. 查 [[life/stack/workspaces|Workspaces]]，确认应该进入哪个工作区。
2. 查 [[life/stack/capabilities|Capabilities]]（或统一入口 [[life/stack/index|Stack — System Map]]），确认已有 agent、tool、skill、automation 和已知缺口。
3. 用 `yatoro skills index search <keyword>` 或 `yatoro skills index list` 查当前可用 skill；不要在 SOP 里硬编码 skill 列表。
4. 查相关 wiki 页面，确认是否已有 SOP、方法论或历史反例。

检索后再决定动作：

| 类型 | 处理方式 |
|---|---|
| 一次性回答 | 直接回答，不沉淀 |
| 一次性执行 | coordinator 直接执行、验证、汇报 |
| 可复用 workflow | 更新 workflow/template 或自动化脚本 |
| Agent 可复用能力 | 生成或更新 skill |
| 长期知识 / SOP | 更新 Obsidian wiki |
| 个人偏好 / 路由 | 更新 Agent MEMORY 或 capability/workspace map |
| 新能力 / 非平凡实现 | 拉 Builder 执行 |
| 高风险 / 需第二意见 | 拉 Reviewer |
| 组合 | 明确拆分哪些执行、哪些沉淀 |

## 当前 Agent Team

当前 #yatoro 默认只有 3 个 agent，见 [[life/stack/agents|Agents]]：

- @yatoro-workflow-coordinator：默认入口；能直接回答/执行，也负责分诊和沉淀决策。
- @yatoro-builder：只有新能力或非平凡实现才进入。
- @yatoro-code-review：只有需要独立复核、风险检查、命令语义/安全边界判断、或第二意见才进入。

不再保留独立 knowledge-curator。wiki/skill/MEMORY 整理是 coordinator 的能力；必要时由 coordinator 自己完成，或把实现部分派给 builder、风险部分交给 reviewer。

## 何时升级

### 直接处理

满足以下条件时，coordinator 直接做：

- 只是回答问题或总结已有信息。
- 已有 CLI/skill/wiki/repo 能完成。
- 文件改动小、风险低、验证路径清楚。
- 没有跨 agent owner 或外部副作用。

### 拉 Builder

出现以下情况时，拉 @yatoro-builder：

- 要新增或修改 Yatoro 代码、CLI、脚本、agent-kit skill。
- 要实现一个以后会复用的新能力。
- 需要较完整的测试/验证闭环。
- coordinator 自己做会让上下文过重或职责混乱。

### 拉 Reviewer

出现以下情况时，拉 @yatoro-code-review：

- 代码/命令变更有回归风险。
- 涉及 destructive/fix/install、外部副作用、权限边界、数据迁移。
- 需要审查测试覆盖、命令语义、source-of-truth 是否被破坏。
- 用户明确要第二意见。

## 迁移决策

任务结束时问：

1. 这是一次性任务，还是以后会重复？
2. 需要写入项目 README/docs 吗？
3. 需要写入 Obsidian wiki 作为长期 SOP 或方法论吗？
4. 模型以后需要主动调用它吗？如果需要，做 skill。
5. Agent 以后需要记住个人偏好吗？如果需要，写 MEMORY。
6. 如果都不值得保留，只留在 thread 并明确 discard。

Memory 分层规则见 [[life/stack/memory|Memory]]。

## 例子

播客抓取任务：

- 需求：自动抓取播客音频并存到 CDN。
- 判断：现有抓取能力若足够，coordinator 直接执行；若要新增后处理链路，则拉 Builder。
- 实现：`yatoro capture rss <feedUrl>`，RSS adapter 发现音频，runner 下载并上传到 R2/CDN。
- 验证：dry-run + 真实抓取硅谷101最新一集。
- 沉淀：CLI README、Agent MEMORY fast path、`yatoro-capture` skill、Obsidian workflow。
