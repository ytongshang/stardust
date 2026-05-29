---
title: Harness Agent — Multi-Agent 协调机制
type: concept
created: 2026-04-14
updated: 2026-04-14
sources:
  - /Users/rancune/Work/github-open-source/OpenHarness/src/openharness/coordinator/coordinator_mode.py
  - /Users/rancune/Work/github-open-source/OpenHarness/src/openharness/tools/agent_tool.py
tags:
  - agent
  - multi-agent
  - coordinator
  - harness
---

# Harness Agent — Multi-Agent 协调机制

> 上级页面：[[Harness Agent]]  
> 相关页面：[[harness-agent/background-worker|Background Worker 实现]]、[[harness-agent/permission-sync|权限同步协议]]、[[harness-agent/inline-agent|Inline Agent vs Background Worker]]

本文聚焦三个主题：**Coordinator system prompt 设计**、**Task notification 协议**、**并发模式**。  
Worker 进程实现见 [[harness-agent/background-worker|Background Worker 实现]]，权限协议见 [[harness-agent/permission-sync|权限同步协议]]。

---

## 一、核心设计决策

Coordinator 和 Worker 跑完全相同的 `QueryEngine`，差异只在配置层：

| | Coordinator | Worker |
|---|---|---|
| 激活方式 | `CLAUDE_CODE_COORDINATOR_MODE=1` | 无此环境变量 |
| 工具集 | 仅 `agent`、`send_message`、`task_stop` | `WORKER_TOOLS` 常量定义的子集 |
| System prompt | Coordinator prompt（~500 行） | 默认 prompt |
| 结果接收 | `<task-notification>` user-role 消息 | 无 |

**没有独立的 coordinator runtime**。同一套代码，prompt 驱动行为分叉。

---

## 二、Coordinator System Prompt 设计

源文件：`coordinator_mode.py:251`

### 2.1 角色与防过度委派

```
You are a coordinator. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work that
  you can handle without tools
```

**不要把能直接回答的问题委派给 worker** 是第一条约束，防止 LLM 把所有事都推给 worker。

### 2.2 工具使用约束

```
- Do not use one worker to check on another. Workers will notify you.
- Do not set the model parameter.
- Continue workers via send_message to reuse their loaded context.
- Never fabricate or predict agent results — wait for real results.
```

最后一条是 **hallucination 防护**：coordinator 不得在 worker 完成前预测结果。

### 2.3 Continue vs Spawn Fresh

```
send_message（续接）when:
  - Worker has relevant context loaded (files read, research done)
  - Follow-up is a logical next step in the same work stream

agent（重新 spawn）when:
  - Task is clearly independent
  - Worker's loaded context would be a hindrance
```

`send_message` 复用已有进程和 LLM 上下文，是对 KV cache 的显式利用。`agent` 产生全新进程，上下文归零。详见 [[harness-agent/background-worker|Background Worker — 重启机制]]。

### 2.4 Meta-prompt：Worker prompt 写作指南

Coordinator prompt 内嵌了写 worker prompt 的指导：

> Never delegate understanding. Write prompts that prove you understood:  
> include file paths, line numbers, what specifically to change.

这是一个递归的 prompt engineering：用 prompt 约束 coordinator 写出高质量的 worker prompt。

### 2.5 Simple 模式

`CLAUDE_CODE_SIMPLE=1` 切换为精简 worker 工具集：

| 模式 | Worker 工具 |
|------|------------|
| 标准 | `bash, file_read, file_edit, file_write, glob, grep, web_fetch, web_search, task_create, task_get, task_list, task_output, skill` |
| Simple | `bash, file_read, file_edit` |

---

## 三、Task Notification 协议

### 3.1 XML 格式

```xml
<task-notification>
<task-id>researcher@analysis</task-id>
<status>completed</status>        <!-- completed | failed | killed -->
<summary>分析完成，发现 2 处问题</summary>
<result>
  ## 分析结果
  ...
</result>
<usage>
  <total_tokens>48293</total_tokens>
  <tool_uses>12</tool_uses>
  <duration_ms>45123</duration_ms>
</usage>
</task-notification>
```

`<result>` 和 `<usage>` 可选。`<task-id>` 直接作为后续 `send_message` 的 `to` 参数。

### 3.2 "假用户消息"注入模式

Task notification 以 **user-role message** 注入 coordinator 的 message list：

```
[system: coordinator prompt]
[user: "修复 CI 失败"]
[assistant: 已启动 researcher 和 scanner...]
[user: <task-notification>researcher 完成...</task-notification>]  ← 注入
[user: <task-notification>scanner 完成...</task-notification>]    ← 注入
[assistant: 综合两个结果...]
```

优势：无需额外 notification API，利用 LLM 原生对话结构；coordinator 可在同一轮处理多个 worker 结果。

Coordinator prompt 明确提醒 LLM：

> Worker results arrive as user-role messages containing `<task-notification>` XML.  
> They look like user messages but are not. Distinguish them by the opening tag.

### 3.3 注入的实际路径

Python 后端是被动的（发出 `tasks_snapshot` 事件），TypeScript 前端是主动的（检测 task completed → 格式化 XML → `submit_line`）。`format_task_notification()` / `parse_task_notification()` 在 Python 侧提供，调用者在前端。

Coordinator 也可主动用工具轮询（无 UI 场景）：

```python
task_get(task_id)      # 查 status / ended_at / return_code
task_output(task_id)   # 读 .log 文件内容（tail 12000 bytes）
task_list()            # 列所有任务
```

---

## 四、并发模式

### 4.1 工具调用层并发

`QueryEngine.run_query()` 遇到多个 tool_use 时：

```python
results = await asyncio.gather(*[
    _execute_tool_call(tc) for tc in tool_calls
])
```

Coordinator 在一次 LLM 响应中调用多次 `agent`，spawn 操作并发执行。

### 4.2 Fan-out / Fan-in 典型流程

```
Turn 1：
  coordinator → [agent] spawn researcher   并发
               [agent] spawn scanner      ↗

Turn 2～N（coordinator 轮询）：
  [task_get: researcher] → running
  [task_get: scanner]    → running
  ...
  [task_get: researcher] → completed
  [task_get: scanner]    → completed
  [task_output: researcher] → 读取结果
  [task_output: scanner]    → 读取结果

Turn N+1：
  coordinator 综合结果
  → [send_message] researcher（续接，复用上下文）
  → [agent] spawn verifier（新任务，独立进程）

Turn N+2：验证完成，向用户汇报
```

### 4.3 Scratchpad：Worker 间直接共享

Coordinator prompt 可配置 `scratchpad_dir`，所有 worker 可读写，无需 coordinator 中转：

```
Scratchpad directory: /tmp/agent-scratchpad/
Workers can read and write here without permission prompts.
```

适合 researcher → implementer 之间传递中间产物（研究报告、patch 文件等）。

### 4.4 Team 广播

```python
registry.send_message(team_name, broadcast_text)
```

`TeamRegistry`（内存，coordinator 进程内快速查找）和 `TeamLifecycleManager`（磁盘 `team.json`，跨进程可见）双层管理。

---

## 五、等待结果的代价：轮询 vs Push

### 5.1 当前实现：LLM 驱动的主动轮询

`QueryEngine` 的循环完全由 LLM 输出驱动，没有任何代码层面的定时器或回调：

```
LLM 推理 → [tool_use: task_get]
代码执行   → {status: "running"}
LLM 推理 → [tool_use: task_get]   ← LLM 自己决定再查
代码执行   → {status: "running"}
...
LLM 推理 → [tool_use: task_get]
代码执行   → {status: "completed"}
LLM 推理 → [tool_use: task_output] ← LLM 决定读结果
```

"等待"不是代码阻塞，而是 LLM 不断消耗推理轮次。

### 5.2 轮询对 Context 的影响

每轮 `task_get → {status: "running"}` 都往 coordinator message history 追加两条消息：

```
[assistant: tool_use: task_get]
[user:      tool_result: {status: "running"}]
```

Worker 跑 5 分钟、每轮查一次，就是十几条纯噪声。后果：

| 问题 | 机制 |
|------|------|
| **Token 消耗** | 每次 LLM 推理重新 encode 全部历史 |
| **KV cache 失效** | message history 每轮增长，prefix 不稳定 |
| **Auto-compact 触发** | 噪声消息累积加速 context 压缩，可能丢失早期重要信息 |
| **LLM 注意力稀释** | 大量重复的 `{status: "running"}` 干扰真正重要内容 |

### 5.3 System Prompt 的缓解尝试

Coordinator system prompt 有一条约束：

```
- Do not use one worker to check on another. Workers will notify you.
```

意图是让 coordinator 不要频繁轮询，等 notification 自动到达。但 LLM 很难严格遵守"什么都不做就干等"——在没有外部打断的情况下，它几乎必然会主动调 `task_get`。

### 5.4 根本解：Push 机制（设计意图，尚未实现）

干净的方案是 push：worker 完成时主动注入一条 `<task-notification>` user message，coordinator 什么都不需要轮询，context 里只有一条有意义的结果：

```
[user: <task-notification><status>completed</status><result>...</result></task-notification>]
```

这正是 coordinator system prompt 描述的设计目标，`format_task_notification()` 也已在 Python 侧实现。但当前代码中没有任何路径自动触发它——`_watch_process()` 无回调，前端 `tasks` state 只用于显示，没有 `task completed → submit_line` 的逻辑。这是一个明确的实现缺口。

---

## 六、开放问题

1. **Coordinator context 膨胀**：轮询产生的噪声消息 + task-notification 累积，auto-compact 如何保留关键通知同时压缩？
2. **部分失败处理**：fan-out 中某 worker 失败，prompt 无明确 retry 规范，依赖 LLM 自主判断。
3. **跨 session 持久化**：team.json 在 coordinator 退出时清理，未完成的 worker 状态无法恢复。
4. **Worker 超时通知**：`max_turns` 耗尽后 worker 退出，coordinator 如何感知？
5. **Push 机制缺口**：`<task-notification>` 自动注入未实现，coordinator 当前只能主动轮询，参见 §5。

---

## 参考

- 源码：`src/openharness/coordinator/coordinator_mode.py`
- 源码：`src/openharness/tools/agent_tool.py`
- 相关：[[harness-agent/background-worker|Background Worker 实现]]
- 相关：[[harness-agent/permission-sync|权限同步协议]]
- 相关：[[harness-agent/inline-agent|Inline Agent vs Background Worker]]
