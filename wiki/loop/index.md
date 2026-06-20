---
title: Loop Engineering
type: concept-index
created: 2026-06-20
updated: 2026-06-20
sources:
  - https://addyosmani.com/blog/loop-engineering/
  - https://www.langchain.com/blog/the-art-of-loop-engineering
  - https://www.sonarsource.com/blog/loop-engineering-without-verification-is-just-automation/
  - https://www.businessinsider.com/what-are-loops-ai-engineering-tips-2026-6
tags:
  - loop-engineering
  - agent-workflow
  - automation
  - verification
---

# Loop Engineering

Loop engineering 是最近 agent / AI coding 圈里出现的一种工作流工程方法。它不是单纯指 agent 内部的 `while` 循环，而是指：围绕 agent 搭建一套可持续运行的外层循环，让系统能够自动发现任务、分配任务、执行、验证、失败重试、记录状态，并进入下一轮。

一句话：

> Loop engineering = 设计 prompt agent 的系统，而不是手动一轮轮 prompt agent。

## 它解决什么问题

传统 agent 使用方式通常是人工驱动：

```text
人提出任务 -> agent 执行 -> 人看结果 -> 人继续提示 -> agent 再执行
```

Loop engineering 想把这件事改成系统驱动：

```text
触发器 -> 任务发现 -> 上下文装配 -> agent 执行 -> 验证 -> 修复/重试 -> 状态更新 -> 下一轮
```

它的目标不是让单次回答更漂亮，而是让 agent 成为持续工作的生产单元，适合处理 bug triage、CI 修复、依赖升级、文档同步、日志分析、issue 分流、PR 初审等重复、可验证、可回滚的工作。

## 在 agent 流程中的位置

Agent 系统里至少有两层 loop：

### 1. Agent 内部 loop

这是模型运行时的基础循环：

```text
读上下文 -> 推理 -> 调工具 -> 观察工具结果 -> 再推理 -> 判断完成或继续
```

这层属于 agent runtime / harness 的核心部分，通常由 agent 框架实现。

### 2. Agent 外部生产 loop

Loop engineering 主要关注这一层：

```text
定时/事件触发
  -> 从 issue、CI、日志、用户反馈里发现任务
  -> 分配给合适的 agent / sub-agent
  -> 在 sandbox 或 worktree 中执行
  -> 用测试、lint、review agent、规则检查验证
  -> 失败则把反馈喂回 agent 重跑
  -> 成功则开 PR、通知、人审或自动合并
  -> 记录状态，进入下一轮
```

所以 loop engineering 不是替代 harness，而是站在 harness 上面调度它。Harness 让 agent 能安全、可靠地调用工具；loop engineering 让这些 agent 能持续被触发、被检查、被改进。

## 和 prompt / context / harness 的关系

```text
Prompt Engineering
  关心怎么写好单轮指令。

Context Engineering
  关心给模型什么上下文、如何预算和压缩上下文。

Harness Engineering
  关心 agent 如何安全可靠地运行：工具、沙盒、权限、状态、日志、恢复。

Loop Engineering
  关心如何把 agent 接进持续生产流程：触发、分配、验证、重试、状态、退出条件。
```

简单理解：

- Prompt 是一句话怎么问。
- Context 是模型该知道什么。
- Harness 是 agent 怎么跑。
- Loop 是 agent 怎么持续工作。

## 典型实践组件

### Trigger

触发 loop 的来源，可以是：

- 定时任务：每 30 分钟检查 CI、每天早上扫 issue。
- 事件触发：CI failed、PR opened、日志出现异常、用户反馈进入队列。
- 人工触发：用户发起一次批量修复或研究任务。

### Task Discovery

Loop 需要自己找到可做的任务，而不是等人手动描述每一步。常见输入包括：

- GitHub issues / Linear tickets
- CI failed jobs
- error logs / incidents
- dependency alerts
- docs drift
- user feedback / support tickets

### Workspace Isolation

每个任务最好在独立 worktree、sandbox 或 ephemeral environment 中运行，避免多个 agent 改同一份代码互相污染。

### Agent Execution

可以是单 agent，也可以是多 agent：

- coder agent：实现修复或功能。
- reviewer agent：审查 diff、找风险。
- verifier agent：判断是否满足任务意图。
- triage agent：归类 issue、判断优先级。

一个重要经验是：写代码的 agent 不应该完全给自己的结果打分，至少要有另一个 verifier 或 deterministic gate。

### Verification

Verification 是 loop engineering 和普通自动化的分界线。没有验证，loop 只是自动重复 prompt，很容易把错误滚大。

常见验证层：

- deterministic gates：test、lint、typecheck、build、安全扫描、secret scanning。
- semantic verifier：LLM 判断是否真的解决了用户问题、是否误改范围、是否需要人工确认。
- production signals：错误率、延迟、成本、用户行为数据、A/B test 结果。

原则：LLM verifier 可以帮助判断语义，但最终停止条件和发布门槛最好有硬检查。

### Retry / Repair

失败后不要只重跑同一个 prompt，而要把失败原因结构化反馈给 agent：

```text
测试失败 -> 摘要失败日志 -> 定位相关文件 -> 要求 agent 做最小修复 -> 再跑验证
```

需要设置最大重试次数、预算上限和人工接管条件。

### State / Memory

Loop 是跨轮运行的，所以必须有耐久状态：

- 当前任务队列
- 已尝试的修复
- 失败原因
- 人工决策
- 下一步
- 预算消耗
- 生成的 PR / artifact 链接

这些状态可以放在 Markdown、issue comment、Linear board、数据库或 event log 中。

### Human-in-the-loop

Loop engineering 不是完全去人化。人的位置通常变成：

- 定义目标和边界。
- 审核高风险变更。
- 调整验证规则。
- 处理 loop 多次失败后的模糊问题。
- 判断业务价值。

## 一个 coding loop 示例

```text
每 30 分钟运行：

1. 读取 GitHub issues、CI failed jobs、错误日志。
2. 选择最高优先级且可自动处理的任务。
3. 创建独立 worktree。
4. coder agent 修复问题。
5. 跑 test、lint、typecheck。
6. reviewer agent 审查 diff 和风险。
7. 如果失败，把失败日志反馈给 coder agent，最多重试 3 次。
8. 如果成功，开 PR 并写清楚变更、测试结果、风险。
9. 更新任务状态，进入下一轮。
```

## 适合的场景

- 有清晰输入源：issue、CI、日志、告警、用户反馈。
- 有明确成功标准：测试通过、lint clean、构建成功、链接可访问。
- 可以隔离执行：worktree、sandbox、权限边界清楚。
- 错误可回滚：PR、feature flag、自动回退。
- 任务重复出现：bug triage、依赖升级、文档同步、release note。

## 不适合的场景

- 成功标准很主观，无法验证。
- 任务高风险，但没有权限隔离和人工审批。
- 没有预算控制，agent 可能无限重试。
- 输入源噪声很大，任务发现本身不可靠。
- 一次性任务，直接 prompt 更快。

## 设计原则

- 先定义退出条件，再设计循环。
- 先做验证，再扩大自动化范围。
- 每轮都要记录状态，不依赖会话记忆。
- 把高风险动作放到人工审批后面。
- 让不同 agent 分工，不让执行者完全自评。
- 限制权限、时间、token、重试次数。
- 优先处理小而可验证的任务。

## 关键区别

Loop engineering 不是：

- 写一个无限循环调用 LLM。
- 用 cron 定时问 agent 同一个问题。
- 把人工流程全部无条件交给 AI。
- 只追求更长时间运行。

Loop engineering 是：

- 有触发源。
- 有任务选择。
- 有隔离执行环境。
- 有验证和退出条件。
- 有失败恢复。
- 有状态记录。
- 有人工接管边界。

## Research Notes

- Addy Osmani 把 loop engineering 描述为 prompt agent 的系统，并强调它位于 harness engineering 之上。
- LangChain 讨论的是 agent 内部 loop 的艺术：模型、工具调用、观察结果、继续/停止判断。
- Sonar 强调没有 verification 的 loop engineering 只是 automation，验证是控制 agent 风险的核心。
- Business Insider 报道了 Claude Code / OpenAI 工程师对 loops 的说法：未来重点不是手写 prompt，而是设计 loop 去 prompt coding agents。

