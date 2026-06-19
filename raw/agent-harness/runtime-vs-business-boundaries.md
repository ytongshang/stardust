# Agent Harness: 业务路由与通用运行时边界

本文拆分 `uxarts-agent` / `uxa-center` 里两类容易混在一起的概念：

- 业务路由概念：只在 Superun/GetUI/UIDraft/Memory/Sub-agent 业务里有意义。
- 通用 harness 概念：任何 agent 产品运行时都可能需要。

目标不是重写现有系统，而是给后续压缩抽象时画边界：保留业务词，但不要让业务词成为底层 runtime API。

## 1. 总原则

`mode`、`stage`、`sub_agent_type`、`wikiAction`、`architecturePlanApproved` 这类词，是业务路由输入。它们可以参与决定本轮怎么跑，但不应该直接散落到 agent loop、tool runner、workspace、compaction、lifecycle 里。

更合理的形态：

```text
BusinessRoute
  -> RunSpecBuilder
  -> RuntimeProfile
  -> AgentRunner / ToolRunner / Workspace / Center lifecycle
```

业务层保留熟悉的词，runtime 层只看通用能力：

- 输入如何转换。
- 哪些上下文在什么时机注入。
- 哪些 prompt block 生效。
- 哪些工具可用。
- 哪些 workspace 写入被允许。
- 历史如何压缩和恢复。
- run 如何 heartbeat、pending、retry、complete。

## 2. BusinessRoute：业务词停在这里

`BusinessRoute` 代表“这是谁的哪种业务流程”，它不是底层执行策略。

典型字段：

```python
@dataclass
class BusinessRoute:
    product: str | None
    mode: int
    stage: str | None
    is_remix: bool
    parallel_index: int | None
    sub_agent_type: str | None
    session_extra: dict
    round_extra: dict
    main_agent_info: MainAgentInfo | None
```

现有来源：

- `GetuiGenerateRequest.mode`
- `GetuiGenerateRequest.parallel_index`
- `GetuiGenerateRequest.session_extra`
- `GetuiGenerateRequest.round_extra`
- `GetuiGenerateRequest.main_agent_info`
- `AgentCreateParams.stage`

这些字段可以继续存在，但建议只由 `RunSpecBuilder` 解释，不要让下游每层都重新理解业务语义。

## 3. RuntimeProfile：底层真正执行的东西

业务 route 编译成 runtime profile。profile 只表达通用 harness 维度。

```python
@dataclass
class RuntimeProfile:
    model_profile: ModelProfile
    agent_profile: AgentProfile
    input_profile: InputProfile
    prompt_profile: PromptProfile
    tool_profile: ToolProfile
    workspace_profile: WorkspaceProfile
    compaction_profile: CompactionProfile
    lifecycle_policy: LifecyclePolicy
    observability_profile: ObservabilityProfile
```

这不是要一次性落所有类。第一步可以只是把现有选择结果记录成一个只读对象，保持行为不变。

## 4. 输入层：InputProfile

`user_input_creator` 不是单一概念，它混了原始输入解析、业务上下文、static context、system-reminder、附件和压缩恢复。建议拆成少数稳定子概念。

```python
@dataclass
class InputProfile:
    user_input_transform: UserInputTransform
    init_message_builder: InitMessageBuilder | None
    business_preprocessors: list[BusinessInputPreprocessor]
    context_injection_policy: ContextInjectionPolicy
```

通用概念：

- `UserInputTransform`: raw input/message -> LLM user messages。
- `InitMessageBuilder`: 首轮初始化消息，例如 workspace tree。
- `BusinessInputPreprocessor`: 业务前处理，例如 prototype business handler、uidraft selection preview。
- `ContextInjectionPolicy`: 决定 static/per-turn/post-compact context 怎么注入。

现有代码锚点：

- `uxarts-agent/uxarts/web/modules/getui/data_processing/message_loader.py`
- `uxarts-agent/uxarts/web/modules/getui/task/prototype/prototype_generate_user_input_creator.py`
- `uxarts-agent/uxarts/web/modules/getui/task/memory/memory_wiki_user_input_creator.py`
- `uxarts-agent/uxarts/web/modules/getui/task/uidraft/uidraft_user_input_createor.py`
- `uxarts-agent/uxarts/web/modules/getui/task/async_sub_agent/sub_agent_run_registry.py`

现有重要行为不能丢：

- `MessageLoader` 只在 `process_status == 0` 时把 `raw_input` 转成 LLM message。
- 已处理过的历史 item 直接复用 `raw_message`，避免重复注入 context。
- prototype creator 支持 business handler。
- attachments 会进入 user message，并保存 user documents。
- read-only sub-agent 使用 lightweight transform，不注入 heavy static/per-turn context。
- general-purpose sub-agent 使用 main-like transform，注入 memory/wiki/plugin/per-turn context。

## 5. 上下文注入：ContextInjectionPolicy

`stage` 里有很多实际不是 stage 的东西，而是“某些上下文在某些时机注入”。

建议显式化：

```python
@dataclass
class ContextInjectionPolicy:
    on_first_turn: list[ContextBlockProvider]
    on_every_turn: list[ContextBlockProvider]
    on_after_compact: list[ContextBlockProvider]
    on_sub_agent_start: list[ContextBlockProvider]
```

通用注入时机：

- `on_first_turn`: 首轮项目初始化、workspace tree、memory/wiki/plugin snapshot。
- `on_every_turn`: plugins/secrets/base_h5_url/miniprogram/file_changes、架构规划每轮提醒。
- `on_after_compact`: summary 后重新注入 static context。
- `on_sub_agent_start`: 子 agent 首轮复制过来的 workspace/context。

现有代码锚点：

- `prototype_generate_user_input_creator.build_static_context`
- `prototype_generate_user_input_creator._build_per_turn_context`
- `conversation_context_window.handle_compact_new_user_content_list`
- `conversation_context_window.handle_create_new_user_content_list`
- `async_sub_agent/general_purpose.py`
- `async_sub_agent/common.py`

分层判断：

- memory/wiki/capability/project tree 是通用 context block。
- architecture planning 文案是业务 block。
- “首轮注入”和“压缩后重新注入”是通用时机。
- “stage4 architecture planning 未确认”是业务条件。

## 6. PromptProfile：prompt block 不是 stage

`stage1/2/3/4` 可以决定选哪些 prompt block，但底层不要认识 stage。

通用概念：

- role block
- workflow block
- tool usage policy block
- safety/injection block
- coding style block
- language/tone block
- product-specific block

业务层可以说“prototype stage3 使用 demo workflow”，但 runtime 层只接收已经选好的 prompt block 列表。

这样做的好处是：同一个 prompt block 可以被 prototype、sub-agent、uidraft 复用；业务阶段只负责选择，不负责执行。

## 7. ToolProfile 与 Policy

工具裁剪现在混合了业务流程和通用权限。建议拆成：

```python
@dataclass
class ToolProfile:
    base_tools: list[ToolRef]
    dynamic_tool_sources: list[ToolSource]
    exclusions: list[ToolExclusion]
    policies: list[ToolPolicy]
```

通用概念：

- tool registry
- dynamic remote tools
- tool exclusion
- tool permission policy
- tool argument validation/repair
- interrupted tool
- sequential/parallel execution
- tool error normalization

业务条件：

- prototype stage1 精简工具。
- prototype consult readonly。
- plan mode 只允许写 `internal/plans/plan.md`。
- architecture planning 未确认只允许 README。
- sub-agent 删除 main-only tools。
- stage4 才开放 Supabase data tools。

现有代码锚点：

- `prototype_tools_builder.py`
- `sub_agent_tools_builder.py`
- `uxarts/core/agent/types/tool.py`
- `uxarts/core/agent/tools/tool_runner.py`

不能丢的行为：

- plugin manifest tools 动态合并。
- skill materialize 到 `.superun/skills`，不一定作为专门工具暴露。
- plan mode 的 editor 硬拦截优先于 prompt 软提醒。
- sub-agent 查询 main session plugin state。
- `ToolInterrupted` 进入 pending/resume 机制。
- tool args 有 JSON repair / nested JSON-like string 修复 / Pydantic validation。
- 同一轮并发写同一文件要被拦截。

## 8. WorkspaceProfile

workspace 不是普通文件系统。它包含产品 artifact、attachment、snapshot、wiki memory、sub-agent workspace、deploy/e2e/sandbox。

通用概念：

- load workspace
- read/write/delete artifact
- enforce write policy
- save attachment
- snapshot/tombstone
- overlay memory
- merge sub-agent output
- expose sandbox/deploy/e2e capability

业务条件：

- uidraft 的 selected html/svg/page resources。
- prototype 的 framework/template/design system。
- wiki ingest 只加载 wiki codebase。
- sub-agent 从 main workspace 派生。

现有代码锚点：

- `getui_context.py`
- `codebase_loader.py`
- `codebase.py`
- `startup_resources/base.py`
- `startup_resources/loaders.py`

建议边界：未来可以有 `WorkspaceBackend` 协议，但第一步只包住现有 `GetuiContext.codebase/codebase_loader/text_editor`，不要迁移存储。

## 9. CompactionProfile

压缩不是简单 summarization。它包括触发、标记、summary、恢复、static context 重新注入。

通用概念：

- token threshold
- needCompress
- summary run
- message trim item
- compact boundary rehydrate
- within-turn continue prompt
- post-compact static context

业务条件：

- general-purpose sub-agent 压缩后重新注入 full static context。
- read-only sub-agent 压缩后只保留 summary。
- wiki memory 是否参与 static context。

现有代码锚点：

- `GetuiGenerateTask.check_has_exceed_conversation_context_threshold`
- `conversation_context_window.py`
- `getui_summary_task.py`
- `AgentCompleteMessageListener.MarkCompressStatus`

不能丢的行为：

- `compressed` 是粘性的，但不能导致每一轮重复注入 first-turn static context。
- compact boundary 才重新注入 static context。
- summary 后要附当前项目结构。
- mode 13 的 post-compact context 由 sub-agent run spec 决定。

## 10. LifecyclePolicy

center 侧是通用 agent 产品运行时，不是业务杂项。它应该被明确看成 lifecycle orchestrator。

通用概念：

- enqueue / dequeue
- distributed lock
- heartbeat
- retry
- pending / resume
- complete
- stop / soft stop
- delayed retry
- min duration
- queue drain/merge
- complete event listeners

业务条件：

- wiki ingest 完成后刷新 sleep/lint timer。
- invitation/token/red envelope 等业务 listener。
- sub-agent completion 回投父 session。

现有代码锚点：

- `uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentApplicationCommandService.java`
- `uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus/AgentCompleteMessageListener.java`
- `uxa-center/uxa-center-service/src/main/java/com/uxarts/uxa/center/service/agent/eventbus/AgentSaveItemListener.java`
- `uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/AgentQueueApplicationService.java`
- `uxa-center/uxa-center-application/src/main/java/com/uxarts/uxa/center/application/agent/SubAsyncAgentApplicationService.java`

不能丢的行为：

- `doRun` 先 heartbeat，再进全局队列。
- 队列 key 有 session 级和 round 级：`sessionId#replyMessageId`。
- 重试前 `fillIncompleteToolCallOutput`。
- failed complete 可进入 delay retry。
- successful complete 可能受 `minDuration` 延迟。
- interrupted tool 不 complete，而是 PENDING 后等待 `batchReplyToolCall`。
- complete event 会触发 git commit、chat queue drain、compress、sub-agent 回投等。
- stop/softStopRound 必须清理 session key 和 round key。

## 11. ObservabilityProfile

观测不应只靠日志。现有 `RunItem` 已经接近 trace schema。

通用概念：

- run item trace
- tool trace
- token usage
- task monitor
- fake item report
- status/skills background messages
- failure taxonomy

业务条件：

- 飞书通知文案和接收方。
- red envelope/status/skills message 这些产品任务。

现有代码锚点：

- `RunHooks`
- `GetuiRunHooks`
- `TraceContextData`
- `TaskMonitor`
- `GetuiRunner._run_background_tasks`

建议：未来 eval/replay 可以直接基于 `RunItem + RuntimeProfile`，而不是重新发明一套日志格式。

## 12. 已经存在的局部好抽象

`async_sub_agent/sub_agent_run_registry.py` 是一个很好的信号：它已经把某个业务类型编译成：

- agent_name
- user_message_transform
- init_messages
- post_compact_context_blocks
- startup_resource_loader

这基本就是 `RunSpecBuilder` 的局部版本。后续可以从这里推广，而不是从零设计。

`startup_resources/base.py` 也已经把启动资源分成稳定 bundle 与 loader params：

- `StartupResources`
- `StartupResourceLoadParams`
- `StartupResourceLoader`

这说明“业务类型选择策略，loader 返回统一 shape”是可行的。

## 13. 建议的压缩顺序

第一步：只读 `RuntimeProfile`

- 在 `GetuiGenerateTask.start()` 里构造 profile。
- 不改变现有调用，只把当前选择结果记录出来。
- profile 先包含 mode/stage 解释后的 model、agent creator、user transform、init builder、startup loader、tool profile 名称。

第二步：抽 `ContextInjectionPolicy`

- 把 static/per-turn/post-compact 的注入时机显式化。
- 先复用现有函数：`build_static_context`、`PerTurnContextBlocks.common_blocks`、sub-agent static builders。

第三步：抽 `ToolProfile`

- 让现有 `create_prototype_tools_func` 和 `sub_agent_tools_builder` 先产出 profile，再由 profile 生成 tools。
- 保留所有现有 exclude 和 editor policy。

第四步：抽 `LifecyclePolicy` 文档/状态图

- 先不改 Java 代码。
- 把 `doRun/heartbeat/pending/complete/retry/stop` 的 invariant 写清楚。

第五步：抽 `WorkspaceBackend`

- 最后做，因为它牵涉 artifact 存储、发布、sub-agent merge、wiki overlay，风险最大。

## 14. 判断一个概念该放哪

可以用这几条判断：

- 如果换一个 agent 产品也需要，它是 runtime 概念。
- 如果只有 Superun/GetUI/UIDraft 业务懂，它是 business route 或 business block。
- 如果它描述“什么时候注入”，它属于 context injection policy。
- 如果它描述“能不能做”，它属于 tool/workspace policy。
- 如果它描述“这轮怎么恢复/完成/重试”，它属于 lifecycle/compaction。
- 如果它只是一个具体文案，不要抽成类，先作为 prompt block data。

最终目标：

```text
业务词保留在上层，runtime 层只执行通用 profile。
现有行为完整映射后，再重构代码。
```
