# uxarts Agent Harness 模块地图

本文按常见 agent harness 模块，对 `uxarts-agent` 与 `uxa-center` 的现有实现做一个概念级分类。目标不是重复代码细节，而是给后续看外部 agent/harness 项目时提供一把尺子：外部项目如果只覆盖这里已有的概念，就不必重点讲；如果提供了更好的抽象、边界、eval 或失败恢复机制，再深入。

## 1. 总体分层

`uxa-center` 是状态机和持久化中枢；`uxarts-agent` 是无状态执行体。一次用户请求在 center 创建 round、排队、调度到 Python；Python 跑 LLM 循环，所有产物、心跳、工具挂起、完成状态都回调 center。

核心链路：

```text
center chat
  -> 创建/读取 session、user message、agent message
  -> doRun(sessionId, replyMessageId)
  -> doCallAgent / StartRunRequest
  -> uxarts-agent /getui/generate
  -> GetuiGenerateTask.run()
  -> GetuiRunner.run()
  -> AgentRunner.run()
  -> saveItem / heartbeat / completeMessage 回调 center
```

主要代码锚点：

- `uxarts-agent/uxarts/web/modules/getui/task/getui_generate_task.py`
- `uxarts-agent/uxarts/web/modules/getui/task/getui_runner.py`
- `uxarts-agent/uxarts/core/agent/agent_runner.py`
- `uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentApplicationCommandService.java`
- `uxa-center/uxa-center-infrastructure/src/main/java/com/uxarts/uxa/center/infrastructure/external/agent/StartRunRequest.java`
- `uxarts-agent/docs/getui/center_agent_flow/`

## 2. Agent 执行循环

`AgentRunner` 是 Python 侧的通用 runner。它负责把 `UxAgent`、历史 `RunItem`、tools、handoffs、guardrail、memory、run_hooks 组合成多轮 LLM loop。

核心概念：

- `UxAgent`: agent 定义，包含 `instructions`、`model_name`、`tools`、`handoffs`、`transfer`、`output_guardrail`、`memory_config`。
- `SingleStepRunInput`: 当前 turn 的 system、messages、delta_run_items、run_items。
- `SingleStepResult`: 一次模型调用 + 工具执行后的结果。
- `NextStep*`: runner 的状态转移，包含 final output、run again、handoff、transfer、interrupted。
- `RunResult`: 一整轮 agent run 的输出，含 `next_run_items`，供下一轮重建历史。

Runner 流程：

```text
_prepare_for_first_run
  -> _generate_system_message
  -> loop:
      get_chat_messages
      model.get_response / streamed_response
      _process_model_response
      execute_tools_and_side_effects
      decide NextStep
```

与常见 harness 对照：这是你们的 agent loop runtime，类似 LangChain `create_agent` / LangGraph compiled graph，但你们以 `RunItem` 事件流作为状态载体。

## 3. RunItem 事件模型

`RunItem` 是整个 harness 的事实日志。它不是简单 messages，而是可回放的 agent 事件流。

常见 item：

- `UxAgentStartItem`: agent/system 起点。
- `UxUserInputItem`: 用户输入或系统注入的 user message。
- `UxModelGenerationItem`: 模型输出。
- `UxToolCallItem`: 工具调用。
- `UxToolCallOutputItem`: 工具结果。
- `UxHandoffCallItem` / `UxHandoffOutputItem` / `UxHandoffInputItem`: agent handoff。
- `UxFlowTransferItem`: final output 后的流程移交。
- `UxMessageTrimItem`: 历史压缩结果。
- `UxCustomizeItem`: 业务自定义 item。

`GetuiRunner.run_step_check()` 会扫描历史 run_items，把缺失的 tool call / tool output 补成 fake item，并上报飞书。这和 DeepAgents 的 `PatchToolCallsMiddleware` 属于同类问题，但你们已经更贴近产品历史修复。

关键不变量：

- 每个 tool call 最终都应有对应 output。
- agent_id/item_id 共同表达多 agent 分支结构。
- center 持久化的是 item 流，agent 每轮通过 item 流重建上下文。

## 4. 任务入口与模式路由

`GetuiGenerateTask` 是一次 Python agent run 的产品外壳。它处理资源加载、mode 路由、context 构建、心跳、完成回调、异常映射。

主要职责：

- 读取 `GetuiGenerateRequest`。
- 加载 startup resources：页面资源、integrations、secrets、file_changes、reference_map、wiki memory。
- 创建 `GetuiContext`。
- 根据 `mode` 选择 agent、user message creator、init message creator。
- 首轮注入 model init messages。
- 解析 center 传来的 messages 为 run_items。
- 调用 `GetuiRunner.run()`。
- 结束后 `completeMessage`；异常时映射 errorType/errorMessage。

常见 mode：

- prototype generate/chat/consult: prototype designer agent。
- sub-agent generate: mode 13，按 `main_agent_info.sub_agent_type` 选择子 agent run spec。
- memory wiki generate: wiki ingest/lint 类任务。
- uidraft 系列: UI draft 专用 agent。

## 5. Prompt / Profile / Stage

你们已有一个业务级 profile 系统，只是没有统一叫 profile。它分散在 mode、stage、agent creator、prompt loader、tools builder 里。

主要维度：

- `mode`: generate/chat/consult/sub-agent/wiki/uidraft。
- `stage`: stage1/stage2/stage3/stage4。
- `parallel_index`: 决定生成分支与模型选择。
- `session_extra` / `round_extra`: 决定是否选风格、是否进入开发、架构规划是否批准等。
- `agent_create_params`: 把这些上下文传给 agent creator、instructions、tools builder。

Prompt 结构：

- system prompt 来自 `UxAgent.instructions`，每轮动态计算。
- prototype 侧按模块拼装：role、goal、workflow_spec、execution_environment、tool_usage_policy、memory、tone/language、injection protection、code style 等。
- workflow 按 stage 裁剪，只保留当前与后续阶段，减少 token 和干扰。
- user message 中大量使用 `<system-reminder>` 注入非用户上下文。

可借鉴点：外部 DeepAgents 的 `HarnessProfile` 把 provider/model/prompt/tools/middleware 差异做成声明式对象。你们后续可以把散落的 mode/stage 工具裁剪、prompt suffix、tool description override 收敛成 `AgentRuntimeProfile`。

## 6. 模型层与工具调用解析

模型调用通过 `LiteLLMChatCompletionsModel` / `model.get_response` / `streamed_response` 进入。`AgentRunner._process_model_response()` 负责把模型输出拆成：

- handoff tool call
- function tool call
- unknown tool call
- server-side hosted tool call

值得注意的恢复逻辑：

- Anthropic server-side tools 以 `srvtoolu_` 开头，不作为本地 tool call 再执行，避免 orphan tool result。
- tool_call 的 `reasoning_content` 和 content 会附到 tool call 上，方便后续记录与工具侧使用。
- `FunctionTool.normalize_arguments` 会在保存/执行前规范化参数。

## 7. Tool 系统

工具核心是 `FunctionTool`，由 `function_tool()` 包装普通函数生成 JSON schema、描述、参数验证与调用入口。

工具能力：

- 严格 JSON schema。
- `tool_args_validate` 做 JSON parse、嵌套 JSON-like string 修复、`json_repair` fallback、Pydantic validation retry。
- `failure_error_function` 将工具错误转成模型可读的 tool result。
- `sequence_priority` 支持部分工具顺序执行。
- `ToolInterrupted` 支持工具挂起。
- `ToolResultWithMessages` 支持一个工具返回多条消息与附件。

工具执行：

- 普通工具并行跑。
- 有 `sequence_priority` 的工具按优先级顺序跑。
- interrupted 工具只保存 call，不执行，返回 `RunResultInterrupted`。
- `ToolRunner` 会拦截同一轮对同一文件的并发修改，避免多工具并发写同一路径。

这部分比很多开源 harness 更细，尤其是参数修复、错误回传、并发文件修改防护。

## 8. 工具集裁剪

工具集不是固定的，而是随 mode/stage/context 动态构建。

prototype 主 agent：

- stage1: 精简工具集，偏探索和需求收集。
- stage2: 去掉日志、present choices、feature manage、task management、run_code、E2E、async sub-agent。
- stage3: demo 阶段，排除插件/Supabase/run_code 等真实后端能力。
- stage4: 完整工具集。
- plan mode: editor 层硬限制只允许写 `internal/plans/plan.md`。
- architecture planning: 未批准时只允许写 README，排除运行/部署/插件/图片等执行能力。

sub-agent：

- 使用独立 `SubAgentToolsBuilder`。
- 删除 main-only 工具：SendMessage、TaskStop、PluginEnable、PluginConfigurationModify、SupabaseEdgeFunctionSecretsCreate 等。
- 增加 `AskMainAgent`。
- stage4 才保留 Supabase 数据工具。
- 防 background 递归。

这是你们的 runtime permission/profile 层，很多外部项目只做到“给一组工具”，没有做到这种产品阶段化裁剪。

## 9. Workspace / Artifacts / 文件系统

`GetuiContext` 持有 codebase、loader、text_editor、task_manager、publish_status 等运行态。文件和产物不只是本地文件系统，而是产品 artifact 体系。

核心对象：

- `CodebaseLoader`: 加载历史产物、wiki codebase、附件。
- `Codebase`: artifacts 的内存视图与更新/删除/发布能力。
- `TextEditor`: 文件编辑入口，支持 allowlist/deny message 等访问控制。
- center 附件系统：MySQL 存索引，HBase 存内容，rowKey 内容寻址；round 视图由快照 + 增量 + tombstone 组成。

与 DeepAgents `StateBackend` 对比：

- DeepAgents 的 `StateBackend` 是 SDK 内部虚拟文件系统。
- 你们的 workspace 是产品级 artifact backend，带保存、发布、快照、复制父工作区、sub-agent 独立工作区、merge artifacts。

可借鉴点：可以抽一层更明确的 `WorkspaceBackend` 协议，让文件工具、sandbox、artifact merge、eval 共享统一边界。

## 10. Memory / Context Compaction

你们同时有短期上下文压缩和长期 wiki memory。

短期压缩：

- token usage 超过阈值后，agent complete 时 `needCompress=True`。
- center 触发压缩任务。
- 历史中出现 `UxMessageTrimItem`。
- 下一轮由 `handle_compact_new_user_content_list()` 把 summary、当前项目结构、post-compact static context 插入 user message。
- 轮内压缩用 `handle_create_new_user_content_list()` 创建继续执行的 user message。

长期 memory：

- wiki memory 由 center scope 表解析 wiki session。
- Python 拉取 wiki attachments。
- 覆写到 `.superun/memory/`。
- 首轮和 compact 后会重新注入 static context。

这套已经覆盖“压缩摘要 + 状态快照 + 继续执行”。外部项目若只是普通 summarization，不算新东西。值得关注的是更好的 recall/eval 或 summary quality 评估。

## 11. Sub-agent / Background Agent

你们有两类子任务：

- inline subagent: `AgentTool` 调 handler，同步返回结果。
- background subagent: 独立 mode 13 `GetuiGenerateTask`，有自己的 session/context/workspace，工具立即返回 agent_id，完成后异步回投主 session。

background 的关键设计：

- background 不是进程内 subagent，而是完整独立 run。
- `TaskUpdate(owner=agent_id)` 由模型把任务和子 agent 绑定。
- `SendMessage` 主到子。
- `AskMainAgent` 子到主，走 interrupted tool + center pending 续跑机制。
- `TaskStop` 根据 owner 找 agent_id，停止对应子 run。
- 有 fanout limit，默认 5。
- launch 使用 tool_call_item_id + sub_tool_call_item_index 做幂等键。
- 子 agent 防递归，不继续开 background general-purpose。

这比 DeepAgents 的同步 `task` 更贴近产品并发执行。外部项目只有在“后台 agent 生命周期/冲突合并/多 agent observability/eval”更强时才值得深挖。

## 12. Interrupt / HITL / Pending

interrupt 型工具是你们的人机协作与 agent-agent 协作底座。

流程：

```text
agent 产生 interrupted tool call
  -> Python 只保存 UxToolCallItem，返回 RunResultInterrupted
  -> center message.status = PENDING(4)
  -> 外部 batchReplyToolCall 写 UxToolCallOutputItem
  -> center status 回 RUNNING
  -> doRun 续跑
```

用于：

- 人工确认。
- `AskMainAgent` 子 agent 请求主 agent。
- 后端自动审批/超时失败。

这对应一般 harness 里的 HITL/approval/resume，但你们已经和 center 状态机整合得比较深。

## 13. 调度 / 队列 / 心跳 / 完成

center 是 run 调度器。

两类队列：

- 全局并发队列：`ZSetQueueHelper`，key 类似 `getui_{env}`，控制正在运行的 round 数。
- 会话消息队列：`ChatQueue`，处理用户连发、join、合并、round 完成后 drain。

心跳：

- Python `GetuiGenerateTask` 注册 heartbeat。
- center 更新 session/message heartbeat。
- 30s 无心跳，`needRetry()` 允许重调度。

完成：

- Python 调 `completeMessage(success, errorType, needCompress)`。
- center 设置 END/EXCEPTION_END。
- 离开 running queue。
- 触发 CompleteMessageEvent，继续队列、压缩、wiki timer、sub-agent 完成回投等。

这个模块是产品级 agent harness 必需，但多数 SDK 项目不覆盖。

## 14. Observability / Hooks

观测主要通过 `RunHooks`、tracing、日志和飞书报告实现。

Python 侧：

- `RunHooks` 统一落 agent_start、user_input、model_generation、tool_call、tool_output、handoff、trim 等 item。
- `TraceContextData` 在 tool 执行时记录 tool_call_item_id、agent_name、agent_id、agent_memory。
- `agent_run_stats` 统计模型调用、工具耗时、错误类型。
- `TaskMonitor` 长时间/单步超时飞书提醒。
- fake tool item、status message、skills message、red envelope judge 等后台观测/辅助任务。

center 侧：

- saveItem event listener。
- CompleteMessageEvent/StopSessionEvent。
- sub-agent ask/main/completion listeners。
- queue/job 超时处理。

主要代码锚点：

- `uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus/AgentSaveItemListener.java`
- `uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus/AgentCompleteMessageListener.java`
- `uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentQueueApplicationService.java`
- `uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/ZSetQueueHelper.java`
- `uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/SubAsyncAgentApplicationService.java`

可借鉴方向：把 observability 从“日志 + item”进一步结构化成 eval trace schema，用于自动分析失败根因。

## 15. Capability / Plugin / Skill

capability 系统把 plugin/skill 从静态工具扩展为 per-session 动态能力。

关键点：

- `CapabilitySnapshot` 挂在 `GetuiContext` 生命周期内。
- 每轮拉 plugin catalog、skill catalog、plugin state。
- skill 按版本增量 materialize 到 `.superun/skills/<id>/SKILL.md`。
- static context 给模型 capability 摘要，不把 skill body 全塞 prompt。
- plugin enabled 后，动态合并远程 manifest tools。
- sub-agent plugin 状态查 main session，skill 可全量安装到自己的 session。

这和常见 `SkillsMiddleware` 类似，但你们已经和产品 plugin state、skill materialization、remote tools 集成。

## 16. Safety / 权限 / 失败恢复

当前 safety 主要通过工具裁剪、editor allowlist、center 状态机和错误回传实现。

已有机制：

- plan mode 只能写 `internal/plans/plan.md`。
- architecture planning 未批准只能写 README。
- stage/mode 删除危险工具。
- sub-agent 删除 main-only 工具。
- 文件并发修改冲突拦截。
- tool args 修复失败时返回模型可读错误。
- unknown tool 返回 tool result，不直接 crash。
- UserInterrupted 不二次 complete。
- fake tool call/output 修复历史。
- center complete/stop/queue 清理避免孤儿 run。

可能增强：

- 把这些规则显式声明成 profile/invariant，启动时校验。
- 给每个工具加统一 capability/permission metadata。
- 对 workspace 写权限从 editor allowlist 扩展成统一 policy 层，覆盖所有写入口和 sub-agent。

## 17. Eval / Benchmark / 回归

已有 E2E 工具和若干测试，但从 harness 视角看，eval 还可以更系统化。

已有相关：

- `run_e2e_test` 工具。
- test/getui 下有 tool args、subagent、plan mode、style classifier、prompt builder、capability context 等单测。
- center/agent 的 docs 已记录关键流程。

可补强方向：

- harness 配置回归：同一任务在不同 mode/stage/profile 下比较工具调用、产物、token、失败恢复。
- trace replay：用保存的 run_items 复现某轮 agent 行为。
- sub-agent eval：background fanout、completion 回投、AskMainAgent 超时 fallback、merge artifacts 冲突。
- prompt/tool profile ablation：像 Harness-Bench/AHE 那样测 harness 组件改动，而不是只测模型。

## 18. 与外部项目对比时的筛选标准

后续看外部 agent harness 项目，可以优先问：

1. 它有没有比我们更好的 profile/config 抽象？
2. 它有没有更清晰的 backend/workspace 边界？
3. 它有没有可复用的 trace/eval schema？
4. 它有没有新的失败恢复机制？
5. 它有没有更好的 sub-agent 生命周期、并发、回投、合并策略？
6. 它有没有工具权限和 HITL 的系统化 policy？

如果只是“有 filesystem、tools、subagent、memory、summarization”，这些你们已经都有，而且更贴产品场景，不必作为重点。
