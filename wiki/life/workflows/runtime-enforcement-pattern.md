---
title: Runtime Enforcement Pattern
type: pattern
created: 2026-05-27
updated: 2026-05-28
sources:
  - "Slock thread #life:a361fe23"
  - "@Agent plan-mode policy switching notes relayed by @Yatoro-claude on 2026-05-27"
tags:
  - life
  - workflows
  - agent-runtime
  - enforcement
---

# Runtime Enforcement Pattern

> 用于把 workflow 阶段、暂停、owner、review gate 从“靠 Agent 自觉遵守”升级成“runtime 可执行约束”。当前 #yatoro 流程是 coordinator-first adaptive flow。

## 核心模式

**single mutator + getter-closure**

运行时维护一个唯一状态源，所有 gate 都动态读取它；状态变更只能由受控入口触发，不能暴露成普通 Agent tool。

## 关键不变量

### P0: Safety Critical

1. **state on context**
   - `ctx.workflow_stage` 是唯一 source of truth。
   - 类似 @Agent / superun 中 `ctx.file_access_policy` 的用法。

2. **canonicalize before deny**
   - 任何进入 gate 的输入都先规范化，再做 deny / allow 判断。
   - superun 对应案例：path normalization 必须先于 prefix-only 检查，否则 `/workspace/../raw/x` 可以绕过策略。
   - 映射到工作流：`workflow_stage`、agent handle、thread target、artifact path 都必须先 canonicalize，避免 stage spoofing、display name 漂移、路径绕过。

3. **enforcement on every route**
   - enforcement 不能只放在一个执行路径上。
   - superun 对应案例：只在 Python 端拦截时，Bash 经 sidecar 可以绕过。
   - 映射到工作流：orchestrator gate、tool gate、@-mention spawn、action card、reminder continuation 都要走同一套 stage / pause / ownership 检查。

### P1: Correctness

4. **mutator NOT a tool**
   - 唯一修改 stage 的方法不开放给 Agent。
   - 只能由 Coordinator、checkpoint-pass event、或 runtime policy event 触发。
   - 这是防止 Agent self-elevation 的关键闸门。

5. **getter closure，不缓存**
   - 每个 enforcement point 都用 `() => ctx.workflow_stage` 读取最新状态。
   - gate 不持有 stage 快照，避免中途升级后旧状态继续生效。

6. **two-phase lifecycle**
   - init phase 和 active phase 必须区分。
   - 不要用“默认 stage = direct”硬塞初始化状态；初始化期间可能还没有 thread owner、work order、artifact roots、reviewer gate。
   - 映射到工作流：`uninitialized -> active(direct / implement / review / preserve) -> closed/paused` 比单一等级字段更稳。

7. **cross-process atomic mutator**
   - 如果状态同时存在于 Python field、sidecar、DB、Slock thread metadata 等位置，只能通过一条 mutator 路径更新。
   - 任一写入失败必须抛 exception，不允许部分成功后继续执行。
   - 映射到工作流：stage、owner、pause flag、artifact locks 要么一起更新成功，要么整体失败。

### P2: Maintainability

8. **mechanism vs business policy**
   - runtime engine 保持 generic：stage store、mutator、getter closure、gate、metrics。
   - 具体业务策略单独放：什么时候直接回答、什么时候需要 Builder、什么时候需要 Reviewer、哪些 action card 需要人点确认。
   - 避免把“播客抓取”“代码实现”“wiki 沉淀”等业务规则写死进 enforcement engine。

## Adaptive Flow 映射

| Stage | Runtime 含义 | Gate 例子 |
|---|---|---|
| direct | 现有知识/能力足够，coordinator 可直接回答或执行 | 可直接回复结果；小范围低风险改动可自测后汇报 |
| implement | 需要新能力或非平凡实现 | Builder 前必须有目标/范围/验收；完成前必须有 verification report |
| review | 实现风险、命令语义、安全边界、测试缺口或需要第二意见 | 禁止单方 close；必须有独立 review |
| preserve | 有复用价值，需要沉淀 | 必须选择 wiki / skill / MEMORY / automation / repo docs / discard |

## 设计含义

- “升级”不是 Agent 自己说了算，而是 Coordinator 或 checkpoint event 改变 `ctx.workflow_stage`。
- Builder / Reviewer / tool gate 只读 stage，不写 stage。
- 所有关键动作都可以写成 gate：
  - 是否允许单 Agent Builder 跑？
  - 是否允许直接改代码？
  - 是否必须先生成 Work Order？
  - 是否必须进入 Reviewer？
  - 是否需要 action card 等人确认？

## Regrets to Avoid

- 标注 transitional code：兼容层、旧字段、临时 fallback 要有类似 `@transitional("v2: remove")` 的标记，否则 v2 清理时很难判断哪些是长期契约。
- Day-1 metrics：从第一版就记录 gate decision、deny reason、stage transition、pause broadcast、mutator failure。事后补观测层比正向写更难。

## 可移植来源

@Agent / superun 后端已有 plan-mode policy mid-session 切换经验：不重启 Agent 即可让 policy 切换。后续实现 Yatoro workflow runtime 时，优先向 @Agent 确认现有实现细节，再移植这个模式。

相关数据点：@Agent 在 commit `632eaa7f` 中补过 canonicalization 边界测试，覆盖 15+ path normalization 场景。这个经验说明 P0 不变量必须先进入设计，而不是等绕过案例出现后再补。
