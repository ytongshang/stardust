# Background Agent — 后端设计方案（uxa-center 侧）

> 状态:**大部分已实现**(launch(含幂等) / main→sub send(reply+排队 append+未投达降级) / stop(清残留 pending) / sub→main AskMainAgent + 单轮兜底 / **完成回投基础版(§3.5,2026-06-10)**);**未实现**:#4 余项(stale 分支 inbox/幂等去重)、更广失败面(#2 余项)、双轮并发窗口(#6)、扇出上限(#12)。配套 agent 侧:见 `background_agent.md`。
> 本文设计 + 记录 uxa-center(Java 后端)如何支撑 background sub-agent。**已落地代码索引见文末 §6。**

## 0. 结论先行

一个 sub-agent = **一个独立的 session(mode 13)**,它的每一次"被发消息"= 该 sub-session 上的一个 round(一次 `GetuiGenerateTask` HTTP 请求;就是 sub-session 自己的前台轮,**不是 background**)。sub-agent 与**父 round** `(parentSessionId, parentReplyMessageId)` 绑定(而非与父 session 整体绑定),以支持将来分支。几乎全部能力(创建/排队/停止/发消息/分支)都能**复用现有 chat / round / queue / stop 基础设施**,新增的只是:一张关联表 + 三个回调入口(launch/send/stop)+ 完成回投。

> **两个 "background" 别混**:
> - Python `AgentTool(background=true)` = "启动一个 sub-agent" 这个动作的名字(本特性的触发器)。
> - uxa-center `ChatRequest.background` = "本轮算不算 session 当前轮" 的前台指针开关(parallel/consult 用)。**本设计不使用它。**

## 1. 复用的现有基础设施(uxa-center)

| 机制 | 位置 | 复用点 |
|---|---|---|
| ~~`ChatRequest.background`~~ | `api/.../ChatRequest.java:44`;`updateSession`:876 | **本设计不用它。** background 只控制"本轮算不算 session 当前轮"(true 时不推进 `lastMessageId/lastReplyMessageId`),是 parallel/consult 的需求。sub-session 内部轮用普通 chat;完成回投到父 session 也要成为父的正常一轮(要唤醒 main),所以也**不用** background。打断/排队是 `isJoin` 轴,与它无关。 |
| chat → round → run | `AgentApplicationCommandService.chat → doRun → doCallAgent → UxAgentServiceClient.startRun → POST /getui/generate` | 创建/驱动一次 agent run 的标准链路。 |
| round 模型 | `Session.Round`,键 `(sessionId, replyMessageId)`(replyMessageId = 用户消息的 messageId) | sub-agent 的每条消息 = sub-session 上一个 round。 |
| 消息树 / 分支 | `Message.preMessageId`(用户消息父指针)+ `Session.userMessageTree()` | 父 session 的分支天然以不同 replyMessageId 区分。 |
| 队列 | `ZSetQueueHelper`,id = `sessionId#replyMessageId`,并发限流 + FIFO 等待队列 | sub-agent 的消息排队、`*` 串行。 |
| stop(软停) | `AgentApplicationCommandService.stop`:completeMessage(success)+ 出队 + `StopSessionEvent` | 停止 sub-agent。 |
| 心跳/超时 | `heartbeat`,30s 无心跳 → `needRetry` | sub-agent run 存活检测,复用。 |
| per-round KV | `SessionRoundExtra` 表,键 `(sessionId, replyMessageId, extraKey)` | 备选的关联存储位置。 |
| 模式 | `SessionModeEnums`,`needsHistory` 控制传全历史还是当前轮 | mode 13 用 **`needsHistory=true`**(已定 #1:子是多轮 session,带历史 + 压缩,和主 agent 一致)。 |
| 回调入口 | `AgentCommandFacade` / `AgentApplicationCommandService`:saveItem / heartbeat / completeMessage / saveAttachment / updateSessionExtra / updateRoundExtra | 在此 facade 上加 launch/send/stop 三个 sub-agent 入口。 |

**注**:现状**没有** parent-child agent 的数据模型(仅 saveItem 有个 `itemFrom=="subAgent"` 的 debug 标记),需要新增。

## 2. 数据模型(需求 #1、#4)

### sub-agent = 独立 session(mode 13)
为什么是独立 session 而非父 session 上的一个 round:sub-agent 自己会有多轮(main 可持续向它发消息 → 多个 round),需要独立的消息历史/队列/round 序列。这与 wiki(mode 12 另起 wiki session)同构,而非与 parallel(同 session 多分支)同构。

### 新增关联表 `agent_sub_async_agent`
```
agent_sub_async_agent
  id                      PK
  parent_session_id       父 session
  parent_reply_message_id 父 round(spawn 它的那一轮)        ← 关键:绑到 round 而非 session
  sub_session_id          sub-agent 自己的 session(= 对外的 agent_id);run 状态去这个 session 查
  sub_agent_type          子类型(sub-designer ...)
  pending_tool_id         子当前挂起的"打断 tool"的 toolId(""=没挂起)   ← 见下
  pending_parent_reply_message_id  处理本次请求的那一轮 main round(主轮执行时 doRun 前同步 bind;"主没回就自动失败"兜底用)
  launch_tool_id          launch 幂等键(发起子的 AgentTool 调用 id);NULL 不参与唯一约束
  created_at, updated_at
  UNIQUE(sub_session_id)                              -- agent_id 全局唯一;send/stop/完成回投按它反查
  UNIQUE(parent_session_id, parent_reply_message_id, launch_tool_id)  -- launch 幂等
  INDEX(parent_session_id, parent_reply_message_id)   -- 列出"本轮的 sub-agents"
  INDEX(parent_session_id, pending_parent_reply_message_id)  -- 父轮 complete 时反查(单轮兜底,带 parentSessionId 防跨 session 撞值)
```

- **不存 run 状态(无 `status`)**:子的运行状态(RUNNING / END / EXCEPTION_END / PENDING)本就在 **sub-session 自己的 session/message 表**上,这里再存一份会两份真相源、会漂移。要看子状态就反查 sub-session。本表只是「关联 + 路由」。
- **(已移除)`cancelled`**:曾用一个布尔标志区分"被 stop vs 自然完成"(两者 message 都落 END)。**2026-06-02 去掉**:重新厘清 stop 语义——**stop 只是打断子当前正在跑的那一轮(END),子 session 并不死,之后 main 仍可 `SendMessage` 续发(正常)**。所以不存在"永久 cancelled"这种态,留个永久标志反而会错误地永久挡住续发。相关判断改为:① 给子发/回填:不按 cancelled 过滤(允许续发);被停那一轮已 END,迟到的 tool 回填会因"非 PENDING"自然 no-op;② 完成回投(§3.5,未实现)需要的"被打断的轮不回投"区分,留到做 §3.5 时按"那一轮是否被打断"处理,不靠这张表的标志。
- **(已移除)`closed`**:曾于 2026-06-03 加 TINYINT 标志做广播去重(`to=*` 只投 `closed=0`)。但广播展开 2026-06-08 已移到 Python(按 **pending/in_progress task 的 owner** 集合,见 §SendMessage),后端 `resolveTargets` **从不读** `closed`——它实质只是个 write-only 位。**2026-06-10 全链路删除**(`closed` 列 + `idx_parent_closed` + 死 mapper `listByParentSessionId` + `markClosed`/`markActive`/`CLOSED_*` + `deliver` 的 reopen 分支 + 整个 `close()` RPC(Java+Python,端到端无调用方)),**行为无变化**。
  - **广播范围 = main 的活跃 task 集合**(单一事实源),不再有后端标志。"已停的子要退出广播"靠 **main 在 TaskStop 后把对应 task 标 completed/deleted**(见 open_questions §T-T1:删 `closed` 后无后端兜底,纯靠模型自觉)。
  - "续发/改派"(`TaskStop(子)` + `SendMessage(子,改做X)`)仍合法:显式 `to=<id>` 本就不受广播集合约束,子 session 常驻可续发,无需 reopen 标志。
- **`pending_tool_id`(关键优化)**:子调 AskMainAgent 挂起(进 PENDING)时,`routeAskMainAgentToParent` 的 `markPending` 把 `toolId`(=`raw_item.call_id`)写到这一行;被回填时 `claimPending`(条件清空)置 ""。
  - 只存 `tool_id` 就够,**不另存 content_id**:`agent_message_content` 有 `idx_st(session_id, tool_id)` 索引,回填时用 `(sub_session_id, pending_tool_id)` 即可走索引直接定位 content,无需冗余字段。
  - 作用:**SendMessage 到来时读这一行即可 O(1) 判 append vs reply** —— 非空 → `batchReplyToolCall(pending_tool_id, message)` 回填;为空 → 入队 append。不必翻 sub-session 的消息找 PENDING tool。
  - 配合"打断 tool 一个一个执行(串行)"的 prompt 约束:任一时刻 `pending_tool_id` 至多一个,无歧义。
- **`pending_parent_reply_message_id`(为什么需要 + 怎么写)**:记录"处理这次子请求的那一轮 main round"。
  - **写入时机**:不是入队时(那时还不知道哪一轮),而是**那一轮主 round 真正执行(走到 chat)、`doRun` 之前【同步】**——`AgentApplicationCommandService.chat` 见 `businessParams.fromSubAgent` 调 `bindParentRoundIfSubRequest → bindParentRound(子, 该主轮 replyMessageId)`。**必须同步且早于 doRun**:eventBus 是 AsyncEventBus,靠 ChatEvent 异步 bind 可能晚于该轮 complete → 子悬挂(M1)。
  - **条件写**(DF2):`bindParentRound` 实际走 mapper `bindParentRoundIfPending`(`WHERE pending_tool_id <> ''`)。若主在该轮 drain 前已早回复、pending 已被 `claimPending` 清空,这条滞后的 bind 不再把残留 replyMessageId 写回去,避免父后续轮 complete 误命中已结清的子。
  - **用途**:主那一轮 **complete** 时(`FailPendingOnParentComplete`)按 `(parent_session_id, pending_parent_reply_message_id)` 反查;若子 `pending_tool_id` 仍非空(主没回)→ 自动失败回填(见 §3.3),保证子**绝不悬挂**。死线绑到这一轮,无需定时器。
- **pending 两字段的清除 + 原子认领(一致性关键)**:清除经 `claimPending`(mapper `clearPendingIfMatch`:`WHERE sub_session_id=? AND pending_tool_id=?` 条件清空,**同时清 `pending_tool_id` 与 `pending_parent_reply_message_id`**),且只在两处、都在**同事务**清除——① reply 回填成功(deliver);② 单轮兜底失败回填。两路径都**先认领后回填**:`clearPendingIfMatch` 返回 1 才回填,否则跳过 → reply 与兜底竞争只有一方成功(H2)。主已回复的子,complete 时反查不到、claim 也抢不到,不会误失败。

- **绑定到 `(parentSessionId, parentReplyMessageId)`**,正是需求 #4:不与 session 整体绑定。将来父 session 出现分支(同 session、不同 replyMessageId),各分支各自拥有 sub-agents,互不串。
- 反向定位父:**单一事实源是 `agent_sub_async_agent` 关联行**——`doCallAgent` 对 mode 13 的 run 反查该行,填 `main_agent_info{session_id, reply_message_id, sub_agent_type}` 透传给 `/getui/generate`(子据此定位父 round、AskMainAgent 路由、完成回投)。**不再往 sub-session `sessionExtra` 写父回指针**(2026-06-08 去掉冗余副本,避免与关联行漂移)。
- **`agent_id` 取值**:直接用 `sub_session_id`(主 agent 存进 `task.owner`、SendMessage 的 `to` 都用它)。

## 3. 三个新入口 + 完成回投

后端独立 RPC 服务 `AgentSubAsyncAgentService`(launch / sendMessage / stop / queryWikiSession / queryMeta)已落地;Python 侧 `web/dubbo/agent_sub_async_agent_service/` Dubbo client 已接线,`launch_background_agent` / `send_agent_message` / `stop_background_agent` / `fetch_sub_agent_wiki_session_id` 原 stub 已换成真调用(`close` RPC 端到端无调用方,已于 2026-06-10 随 `closed` 字段一并删除;见 §2)。

> **子 agent 的 wiki 记忆解析(2026-06-08)**:wiki sessionId 原由后端启动时查 wiki scope 表(`agent_wiki_session_scope`,按 user/org)注入主 agent 的 `session_extra`,Python `fetch_wiki_memory` 读 `userWikiSessionId`。子 agent(mode 13)自身 `session_extra` 不带 wiki id(父链回指针已不写 sessionExtra),故新增 `queryWikiSession(subSessionId)`:后端据子 **反查关联表找父 session**,取父 user/org,**再查 wiki scope 表**返回 user/org 两级 wiki sessionId。**不读父 session_extra**——以 scope 表为单一事实源。Python 侧分层:`GetuiGenerateTask.load_wiki_memory()` 做"主/子"分流(子 → `fetch_sub_agent_wiki_session_id(sessionId)` 经 RPC;主 → 读 `session_extra.userWikiSessionId`),解析出 `wiki_session_id` 后交给纯函数 `GetuiContext.fetch_wiki_memory(wiki_session_id)` 拉附件。

### 3.1 launch(对应 Python `launch_background_agent`)
入参(已定契约):`parentSessionId(=session_id)`, `parentReplyMessageId(=reply_message_id)`, `subAgentType`, `description`, `prompt`, `imageUrls?`, `roundExtra?`(透传给子首轮 `ChatRequest.roundExtra` → `/getui/generate` 的 `round_extra`)
流程(**全部在后端做,不从 agent 传附件**):
1. 事务内新建 sub-session(`mode=13`)+ 写 `agent_sub_async_agent` 关联行(先建关联,见 §5/#5);父链只落关联行,**不写 sessionExtra 回指针**(子 run 由 `doCallAgent` 反查关联行填 `main_agent_info`)。
2. **拷贝父 session 的当前产物到 sub-session**(主要是代码,见下"附件拷贝")。
3. 以普通 `chat`(mode=13)在 sub-session 上建首个 round 并 `doRun`(→ `/getui/generate` 用 mode 13 跑 `SubAgentDesignerAgent`)。**用普通 chat,不要 `background=true`**(background 只是前台指针开关,对独立 sub-session 无意义)。
   - **prompt 以用户输入的形式进**(content,可带 attachments / businessParams),这样到 agent 侧会变成一个 **`UxUserRawInput`**,由 `sub_agent_user_message_creator(raw_input, ...)` **再处理一遍**(拼 system-reminder、附件、首轮 memory/wiki 等),和 mode 6 走 `prototype_user_message_creator` 同构——**不要当成裸字符串直接喂模型**。
   - **格式要完整,字段不能少**:首轮 user 消息要按正常 chat 的标准 item 结构落库——agent 侧 message_loader 会 `UxUserInputItemData.model_validate(item.data)` 校验并取 `data.raw_input`(`UxUserRawInput{type, content, data, attachments}`);且"`raw_message` 与 `raw_input` 不能同时为 None"。**所以走正常 chat 的 user-message 构造路径**(它已生成完整结构),别手搓一个残缺 item——缺 `raw_input`/字段会校验失败或命中报错分支,sub run 直接挂。
4. 返回 `agent_id = sub_session_id`。
> 主 agent 拿到 agent_id 后自行 `TaskUpdate(owner=agent_id)` 关联(已定:关联是模型的活)。
> 顺序要点:**先建 session+关联+拷贝附件,最后才 `chat`**(`chat` 是很多地方共用的入口,拷贝逻辑放它前面、不要塞进 `chat`)。

#### 附件拷贝(父 → 子,launch 时)
- **必须用父 session 的「当前完整态」,不能只用上一个 snapshot**:本轮 agent 可能已经先写了若干文件、再调本工具,只读快照会漏掉这部分。
  - uxa-center 的当前态 = `immutableAttachment`(快照基线)**+** `attachment`(可变增量,含本轮新写的文件)。`AttachmentDomainService.query(...)` 已经把两者合并成"当前视图",**拷贝走这个合并视图**(以父的当前 round / `lastReplyMessageId` 为版本边界),**不要直接读 `immutableAttachmentRepository`**(那只是快照,会漏本轮写入)。
- **exclude 前缀**:加一个可配置的「按 `startsWith` 前缀排除」方法,只拷代码、去掉内部/元数据。默认排除(对齐 Python `excluding_artifact_prefixes` + internal):`internal/`、`compiled/`、`src/ui/svg-assets`(`.superun/memory/` wiki 另由 mode13 的 wiki 注入,不必随附件拷)。清单可调。
- 拷到 sub-session:按合并视图里的 `name → content`(过滤后)写成 sub-session 的附件,作为子的初始工作区。

> **工具划分(简化版,定稿)**:主 agent 侧只两个工具——**SendMessage**(发消息,带 `interrupt` 控制投递时机)和 **TaskStop**(停掉子当前执行的那一轮;执行级、不改 task 状态);子 agent 侧一个**打断 tool**(`ToolInterrupted` 型,sub→main 请求)。不再有 `ReplyToSubAgent`、`delegationRequestId` 那套。
>
> **SendMessage 的 `interrupt` 与 TaskStop 是两个独立概念,别混**:`TaskStop` = 停掉子当前那一轮(清队列 + 软停)——**执行级,不改 task 状态、不等于完成**,子存活可续发;退广播靠 main 显式 `TaskUpdate(completed/deleted)`。`SendMessage(interrupt=true)` = 只打断当前轮再投这条消息,子继续存活、**不清队列**,END→drain 起新一轮跑它。二者共用底层 `stopSubAgentSession`,但语义/落点不同。

### 3.2 main → sub:SendMessage(reply/append 后端自动判,interrupt 控制投递时机)
工具入参:`to(=sub_session_id 或 "*")`, `message`, `summary?`, **`interrupt?`(默认 false)**。

**广播 `"*"` 的展开在 Python 侧做**(2026-06-08):`send_agent_message` 收到 `to="*"` 时,用 `GetuiContext.task_manager` 列出**pending/in_progress 且带 owner 的 task**(2026-06-10 S7:completed/deleted 不广播——广播范围=当前活跃任务集,避免 deliver 的 markActive 把已完成/已停的子重新唤醒),把它们的 `owner`(= background agent id)去重成显式 `targets` 列表;`to` 为具体 id 时 `targets=[to]` **直接透传、不按本地 task 过滤**(2026-06-10 S6:归属校验交后端 resolveTargets,空结果由工具层提示)。后端 RPC 入参因此从 `to` 改为 **`targets: List<String>`**,**不再认识 `"*"`**——广播集合 = main 的活跃 task 集合(单一事实源是 task 列表,不再靠后端 `closed` 过滤)。

`interrupt` 控制"子正忙时这条消息何时生效":**false**=等子当前轮跑完,下一轮再拼上去(普通后续消息);**true**=立即软停子当前轮、让它马上改做这条消息(纠偏/改派/中止当前方向时用)。子空闲或正挂起等回复时本标志无影响——消息即刻生效。**注意它不等于 TaskStop**:子不会被关闭,只是当前轮被打断后立刻处理新消息。

后端 `sendMessage` 对 `targets` 逐个定位、按父 session 校验归属后投递;每个目标**读 `agent_sub_async_agent.pending_tool_id`** 即可 O(1) 二选一:
```
对 targets 中每个 id 找到 agent_sub_async_agent 行(并校验确属当前父 **session**——不卡 spawn 轮,主回复子的 AskMain 是在后续轮发的;不属/不存在的丢弃)：
  ├─ pending_tool_id 非空(子正挂在打断 tool 上)  —— interrupt 在此分支无意义:子已挂起等回复,回填即立刻恢复,无"当前轮"可打断
  │     → reply：低层回填 —— 「batchReplyToolCall(pending_tool_id, toolResult=message) + 清空 pending_tool_id」
  │       放在同一事务原子提交(回填成功但清 pending 失败会整体回滚,不残留),提交后再 doRun 续跑
  └─ pending_tool_id 为空
        ├─ interrupt=true：先 `stopSubAgentSession(子)` 软停当前轮(只这一轮;不清队列),再入队 ↓
        └─ → append：入队 —— 走现成排队 `AgentQueueApplicationService.in(sessionId=子, item={content=message})`：
            · 子还在跑 → 存 `ChatQueue` 行排队,等当前轮(自然完成,或被上面 interrupt 软停)结束后由 drain 拉起(postQueue → chat)；
            · 子已结束(END/EXCEPTION_END) → 立即执行。
          (不用普通 chat 直投：chat 的 else 分支会 `completeMessageWithJoinCascade` 先完成当前轮 = 打断在跑的子。)
```
"如果有打断tool就reply,没有就入队加一句话"——判断全在后端、查一行即可,主 agent 不用选。`pending_tool_id` 由子挂起时写入、回填后清空(见 §2 表)。

> **interrupt 的"先软停后入队"为什么不丢消息**:软停只是发 END 事件、子下次 saveItem 才真停,不是同步立即结束。两种竞态都安全——① drain 在入队提交后才触发 → 消费已提交的排队消息;② 子在入队前已转 END → `in()` 走"已结束立即执行"。子本就空闲时 `stopSubAgentSession` 是 no-op(无可停轮)。`in()` 持 `getBySessionIdForUpdate` 锁原子二选一(排队 xor 立即执行),不会双跑。

> **实现要点(后端落地)**:
> - reply 走 **domain 层** `batchReplyToolCall` / append 的排队消费走 chat —— 都是系统内部 main→sub 投递,**不走前端 facade 的 org/user / 编辑权限 / 风控**(`replyToolForSubAgent` 直连 domain;`AgentQueueApplicationService.in` 的 org/user 仅供其顶部风控,实际 chat 用 session 自身的 org/user)。
> - reply 的「回填 + 清 pending」同事务原子;`doRun` 一律放事务外(与 `chat()` 同序,避免远程 run 回调早于提交)。
> - append 入队后,sub-session 那一轮会经普通 `chat()` 跑起来,会触发 `ChatEvent`(其 `RefreshOtherAfterChat` 对无人订阅的子 session room 发一次 SocketIO,无害);launch 首轮则走专用 `chatForSubAgent` 不发 `ChatEvent`。

**为什么不需要 requestId**:靠 prompt 约束子 agent**打断 tool 一个一个执行(串行)**,任意时刻一个 sub 最多只有**一个**挂起的打断 tool → 后端凭 `to`(subSessionId)就能唯一定位它,无需关联 id。

**`to="*"`**:Python 侧已把 `*` 展开成显式 `targets`(= main 的 pending/in_progress task 的 owner 集合),后端 `resolveTargets` 只按 `parentSessionId` 校验归属、对每个 target 各做一次上述判断/投递,**不再有后端广播分支/`closed` 过滤**。(广播范围 = main 当前活跃 task 集;已停的子退出广播靠 main 把 task 标 completed/deleted。)

### 3.3 sub → main:`AskMainAgent`(子侧打断 tool,`ToolInterrupted`)— 已实现
工具:`tools/agent_message/ask_main_agent_tool.py` 的 **`AskMainAgent(request, summary?)`**(`ToolInterrupted` 型;只进 `SubAgentToolsBuilder`,主 agent 没有)。无 `to`——收件人永远是本子的父/主 agent,后端按 sub-session 父关联解析。
场景:能力只在主 agent(如 **Supabase DDL 只允许主 agent 改**)。**全程不打断父**(请求排队进父循环)。

完整链路(代码):
```
① 子 agent 调 AskMainAgent(ToolInterrupted)
   → sub-run 发 tool_call_item(无 output)→ saveItem(needInterrupt) → center 把 sub round 标 PENDING(4)，post SaveItemEvent
② [SubAgentAskMainListener.RouteAskMainAgent] 命中 SaveItemEvent.isAskMainAgent()（raw_item.name=="AskMainAgent"）
   → 取 toolId(raw_item.call_id) + 参数(arguments: request/summary)
   → routeAskMainAgentToParent：
        a) markPending(subSessionId, toolId)  —— 记下子挂起的打断 tool
        b) 入队到父 AgentQueueApplicationService.in(parentSessionId, item{content=request 原文,
           businessParams={business_type:"sub_agent_request", fromSubAgent:子id, subAgentRequestSummary:summary?}})
           —— 父忙则排 ChatQueue、当前轮跑完再拉起；父空闲则立即跑。【不打断父】
           入队整段 try-catch【DF1】：入队抛错(父不存在/队列异常)→ 立即 replyToolForSubAgent(子, toolId,
             ROUTE_TO_MAIN_FAILED, claim=claimPending) 失败回填,否则子永远卡 PENDING(没人会再触发兜底)。
③ 父侧 prototype_user_message_creator 按 businessParams.business_type 路由到
   handlers/sub_agent_request.py → 注入 <sub_agent_request agent_id="子id"> system-reminder：
   说明"某子已暂停等你，用 SendMessage(to=子id) 回复以唤醒；不做也要回个理由让它 fallback"
④ 主 agent 执行 → SendMessage(to=该子, message=结果)
   → §3.2 reply 路径：后端见该子 pending_tool_id 非空 → claimPending 原子认领 + batchReplyToolCall 回填(同事务) → 子续跑
```
- 主 agent **不需要专门的 reply 工具**:用 SendMessage 回该子,后端按 `pending_tool_id` 自动变回填(§3.2)。
- **串行约束**:prompt 要求子一次只挂起一个打断 tool,避免"主该回哪个"歧义 → 凭 `to`(subSessionId)即可唯一定位。
- **排队不打断**(原 #10 已定并落地):请求经 `AgentQueueApplicationService.in` 进父循环,排在主当前轮后唤醒,不打断主当前轮。

**单轮兜底:主没回就自动失败,绝不悬挂 — 已实现**
锚定方式:**带 businessParams.fromSubAgent 的那一轮主 round 真正执行(走到 chat)时,在 doRun 之前【同步】**把"处理它的主 round"写进关联表;主轮 **complete** 时反查、若子仍挂着就自动失败回填。
```
绑定 [同步,AgentApplicationCommandService.chat,doRun 之前]：
  chat：doChatInTransaction 建轮 → bindParentRoundIfSubRequest(cmd.businessParams, replyMessageId)
     若 businessParams.fromSubAgent 存在 → bindParentRound(子id, 该主轮 replyMessageId)
  → 然后才 doRun。【必须同步且早于 doRun:eventBus 是 AsyncEventBus,异步 bind 可能晚于该轮 complete → 子悬挂】
失败兜底 [SubAgentAskMainListener，父轮两条终结路径都订阅，共用 failPendingByMessage]：
  · 自然完成 → application completeMessage 发 CompleteMessageEvent → FailPendingOnParentComplete；
  · 被 stop → softStopRound 走 domain completeMessage(不发 CompleteMessageEvent)、只发 StopSessionEvent → FailPendingOnParentStop。
  二者都：加载该 agent 消息取 replyMessageId(= 该轮用户消息 id)
    → failPendingForParentRound(parentSessionId=该 session, replyMessageId)：
       按 (parent_session_id, pending_parent_reply_message_id) 反查 agent_sub_async_agent  【M3:带 parentSessionId 防跨 session 撞值】
       对每个【pending_tool_id 仍非空】的子：
         → replyToolForSubAgent(子, pendingToolId, 失败文案, claim=claimPending)  —— 原子认领抢到才回填,子收到失败结果自行 fallback
```
- 死线 = **处理它的那一轮 main round 结束**,无需定时器;子永远会被唤醒,不卡 PENDING。
- 主**在这一轮内**用 SendMessage 回了 → 走 reply 回填(真实结果),`claimPending` 原子清两个字段 → complete 时反查 `pending_parent_reply_message_id` 已查不到、claim 也抢不到 → **不触发兜底**。
- **原子认领(H2)**:reply(deliver)与兜底(failPendingForParentRound)都先经 `claimPending`(mapper `clearPendingIfMatch`:`WHERE sub_session_id=? AND pending_tool_id=?` 条件清空)在同事务内**抢占**,只有抢到者 `batchReplyToolCall`;回填异常与认领一并回滚。被停的子那一轮已 END,迟到/兜底的回填会因"非 PENDING"自然 no-op,无需 cancelled 标志。
- 配套索引:`agent_sub_async_agent` 的 `idx_pending_parent` = 联合 `(parent_session_id, pending_parent_reply_message_id)`(complete 每轮反查走索引)。
- **已知边角**:`pollNextWithMerge` 把同 session 多条排队消息合并成一轮时,只会 bind 其中携带的那个 `fromSubAgent`;若多个子请求合并进同一主轮,另一子这轮不触发兜底(常见单请求不受影响)。主 session 整体被 stop/删除、主轮跑太久等更广失败面仍归 **待定 #2** 余项。

### 3.4 打断 / 停止:用 TaskStop(需求 #2)— 已实现
**复用现成的 TaskStop**(语义=停一个正在跑的 background task;一个 sub-agent 就是一个 background task)。
- 主 agent `TaskStop(task_id = <todo 任务 id>)` → 工具在 **TaskManager 里查该任务的 `owner`(= agent_id)** → `stop_background_agent(agent_id)` → RPC `AgentSubAsyncAgentService.stop`:
  - **语义**:stop = **打断子当前正在跑的那一轮**(END)。子 session **不死**,之后 main 仍可 `SendMessage` 续发(正常)——所以**不置任何永久 cancelled 标志**(已移除,见 §2)。
  - `SubAsyncAgentApplicationService.stop`,**严格按序**:
    1. **`agentQueueApplicationService.clearBySessionId(子)` 清空该子待投的排队 append**——**关键**:停止会触发 `completeMessage`(→`CompleteMessageEvent`)和 `StopSessionEvent`,二者都有监听器去 **drain 队列**(`pollNextWithMerge → postQueue → chat`);若不先清空,排队里的 append 会把**刚被打断的子重新拉起一轮**。必须在第 2 步(发这两个事件)之前清,且**故意分两个事务**(drain 监听器异步读已提交数据,clear 须先独立提交);
    2. `stopSubAgentSession`(软停当前轮);
    3. 软停后重 select,对残留的非空 `pending_tool_id` 走 `claimPending` 清挂起(子曾挂在 AskMainAgent、那轮已 END、tool 再不会被回填)。
    （退出广播不在此处做:无 `closed` 标志,靠 main 把对应 task 标 completed/deleted。）
  - `AgentApplicationCommandService.stopSubAgentSession`(**子专用低层软停**):`completeMessage(END)` + 出队 + post `StopSessionEvent`;**不走 facade `stop` 的 `doCheckEditorAccess`(内部调用、无前端 org/user),也不做 parallel 处理(子无并行)**。sub 下次 saveItem 拿 600010001 自停(=§4 链路)。
    - **与 facade `stop()` 共用私有 `softStopRound(StopRequest, 轮级 key 后缀)`**(completeMessage + 出队 + StopSessionEvent),便于以后同步同一套停止逻辑;**轮级 key 后缀由调用方传**:sub 与 facade 均传 `getReplyMessageId()`(facade 原误用 `getPreMessageId()`,2026-06-10 确认为既有泄漏 bug 并修复,详见 open_questions §facade stop 条目)。
    - **出队 key(sub 侧)= `getReplyMessageId()`**:与 `doRun` 入队 `sessionId#replyMessageId`(= 该轮用户消息 id)一致;为空则不拼,避免 `sessionId#null`。session 级两个 `leaveQueue(sessionId)` 照常。(facade 原用 `getPreMessageId()`,2026-06-10 已确认为泄漏 bug 并修复对齐,见 open_questions。)
- 即:`task_id` 是 **todo 任务 id**(不是 agent_id);agent_id 通过任务的 owner 反查(主 agent 之前用 `TaskUpdate(owner=agent_id)` 关联过)。任务无 owner → 无关联 agent,不停。
- "这个子功能不要做了" = `TaskStop(子)`(打断当前轮)+ 把该 task 标 completed/deleted(退出广播集合);要改做别的 = `TaskStop(子)` + `SendMessage(子, "改做 X")`——显式 send 不受广播集合约束,子常驻可续发。
- 也可由用户在前端点"停止某子任务"→ 带 agent_id 调同一后端 stop。
- ⚠️ 软停的"双轮并发窗口"风险仍在(旧 run 到下次 saveItem 才停),见待定 **#6**。
- 语义:TaskStop = **打断该子 agent 当前正在跑的那一轮**(当前轮 END + 清排队);子 session 存活,之后仍可 `SendMessage` 续发(无"永久停用"态)。

### 3.5 完成回投(sub-agent 跑完 → 通知 main)— ✅ 基础版已实现(2026-06-10)
> 实现:`SubAgentCompletionListener.ReportSubRoundOnComplete` 订阅 `CompleteMessageEvent` → `SubAsyncAgentApplicationService.reportRoundCompletionToParent`。**触发面由事件机制天然圈定**——该事件全系统仅 `doCompleteMessage`(自然终态)一处发;stop 走 softStopRound(domain complete,无 eventBus)只发 StopSessionEvent、子挂 AskMainAgent 不调 complete → **被打断的轮天然不投**,无需 cancelled 标志(下方原设计的"区分被打断"问题就此解决)。**失败终态轮(重试耗尽)也投**(文案带 errorType/errorMessage)。**单一所有者 drain-or-report(2026-06-11 T4)**:回投不再只读 `hasQueuedItems`(那与 generic drainer 并发、有"删队列→建新轮"间隙误判空闲的竞态),而是自己 `pollNextWithMerge`——弹出 main 待投 append → `postQueue` 起新一轮、**本轮不投**(等最后一轮);弹出 null → 队列确实空 → 才投。generic 完成监听器对子 session 跳过 drain(`isSubSession`),drain 与 report 收口同一顺序决策、互斥、无竞态。**本轮仍是子最新一轮才投**(防御 interrupt/显式 send 另起轮)+ **幂等(2026-06-10 T5)**:用 `markReportedIfNot`(关联表 `last_report_message_id` 条件 UPDATE)原子认领本完成轮的回投权,重复 `completeMessage` 回调(Python 重试/心跳超时 watchdog)命中 0 行跳过,父不会被同一轮重复吵;最终文本按时间序取该轮**最后一条有文本的 `model_generation_item`** content 的 `raw_message.content`(字符串或 block 列表拼接;agent_end 不一定存在,故不用);投递复用 AskMainAgent 排队路径(`business_type=sub_agent_report`,不打断父、锚定父当前分支,best-effort)。**产物 diff(2026-06-10;base 口径 2026-06-10 改 immutable)**:回投时比对「子的**不可变基线视图**(launch 时 `copyByRowKeyToImmutable` 冻入子 session 级 immutable 附件的父产物拷贝,经 `immutableOnly` 读出) vs 完成轮视图」的 name→rowKey(排除 internal/compiled/svg-assets/.superun,不读 HBase);**没改不提合并**,改了把 {added/modified/deleted} 清单放 `businessParams.subAgentChangedFiles`。父侧 `handlers/sub_agent_report.py` 包装提示对账 task + (有改动时)列清单、提示主调 **`MergeAgentArtifacts`** 合并(三路合并:base=子不可变基线(`immutableOnly`)/target=主当前/incoming=子当前,沙箱跑 `snapshot_merger_skill`,成功后 ChangeSet 应用回主 codebase;key 暂读 `ANTHROPIC_API_KEY` 环境变量)。**为何用 immutable 基线而非"首轮视图"**:子首轮的写入也落该轮,首轮视图会被污染;immutable 基线只含 launch 冻结的父产物、不被子各轮写入污染(详见 `center_agent_flow/07-attachments.md`)。**#4 余项仍暂缓**:stale/已回退分支 inbox、显式 preMessageId 锚定校验(回投轮级幂等已由 T5 解决,见 open_questions §T-T5)。
> 注意与 §3.3 兜底区分:§3.3 是"子 AskMain 请求、主没回 → 失败回填子";本节是"子一轮跑完 → 通知父"。以下为原设计稿(供 #4 余项参考):
- sub-session 的 round 完成 → 已有 `completeMessage` 回调命中;
- 在 `doCompleteMessage` 里:若该 session 是 mode 13,按 `sub_session_id` 反查 `agent_sub_async_agent` → 拿到 `(parentSessionId, parentReplyMessageId)` → 把 sub-agent 的最终文本(读其最后一条 agent 消息)作为一条 user message **注入父 session**,作为主 agent 的**正常一轮**(要唤醒 main 去处理、用户也看到反应)——**不用 `background`**(我们就是要它成为父会话的当前轮 + 触发 main run);
- **被打断的轮不投**:stop 打断的那一轮也是 END,但它不是"完成",不该把其(被截断的)结果投给父。`cancelled` 已移除,所以这里**需要在做 §3.5 时另外判断"这一轮是否被 stop 打断"**(如:stop 时给该轮打个标 / 回投钩子识别),区分"自然完成 → 投"与"被打断 → 不投"。留待 §3.5 实现。
- 两条相关轴:① 这条注入是**打断**父当前轮还是 **join**(排在父正在跑的轮之后)= `isJoin`,建议 join,别打断用户主线;② 父轮已 move on/分支/结束时怎么落、是否自动 run = 待定 **#4**(锚定分支 tip + active 校验 + 幂等;stale 写 inbox 不自动 run);
- 主 agent 据 agent_id(= owner)对上任务 → `TaskUpdate(completed)`。

## 4. 打断 / 停止机制(已确认,复用现有)

主 agent 现有的"用户 stop"打断机制已查清,**sub-agent 直接复用同一套**,无需新增强杀通道:

```
stop(session, messageId)
  → completeMessage(success=true) → 消息 status = END(1)
       （AgentApplicationCommandService.stop / AgentDomainService.completeMessage）
  → 正在跑的 Python run 下一次 saveItem 调用
       → AgentDomainService.saveItem 检测 status==END
       → 返回 SaveItemResult.fail(600010001 = AGENT_MESSAGE_EXPIRED)
  → Python real_save(getui_run_hooks.py:419) 识别 errorCode==user_interrupted(600010001)
       → raise UserInterruptedException
  → GetuiGenerateTask 捕获(getui_generate_task.py:145) → need_report=False → 优雅中止(不二次 complete)
```

要点:**不是强杀,而是"标记 END + 让下一次 saveItem 拿到 600010001 自爆"**。所以打断的生效粒度 = 下一次 saveItem(通常很快,因为生成过程中频繁 saveItem)。

对 sub-agent:stop / 打断 = 对 **sub-session 的当前 RUNNING round** 调同一个 `completeMessage(END)`,其正在跑的 GetuiGenerateTask 下一次 saveItem 即自爆中止。无需任何新机制。

## 5. mode 13 在 uxa-center 的接入(对齐 wiki/mode12 模板)

1. `SessionModeEnums` 增 `SUB_AGENT_GENERATE(13)`,**`needsHistory=true`**(已定 #1:子是多轮 session,要带自己的历史 + 有界压缩,和主 agent 一致;不是 mode 12 那种无状态)。
2. sub-session 的信息经 `MainAgentInfo`(`session_id` / `reply_message_id` / `sub_agent_type`)带在请求上(Python 侧 `main_agent_info`),供分发与回投使用。
3. `doCallAgent` 对 mode 13 走 `needsHistory=true`(带历史、可压缩)。
4. ~~`completeMessage` 增加 mode 13 的"完成回投"钩子~~ 已实现(2026-06-10):经 `CompleteMessageEvent` 监听器(非 doCompleteMessage 内联),"自然完成 vs 被打断"由事件机制天然区分(stop 不发该事件),见 §3.5。

## 6. 端到端时序(汇总)

```
主 agent(父 session S, 父 round R=replyMessageId):
  AgentTool(subagent_type="sub-designer", background=true) → Python launch_background_agent
    → [uxa-center] launch: 事务内建 agent_sub_async_agent(S,R)->S' + 建 sub-session S'(mode13)
                          → chat(mode13, prompt) on S'(普通 chat,非 background) → /getui/generate 跑 SubAgentDesignerAgent
                          → 返回 agent_id=S'
  → 主 agent TaskUpdate(owner=S')

中途:
  主 → 子:SendMessage(to=S', message)
       子挂在打断 tool(pending_tool_id 非空) → batchReplyToolCall 回填(+同事务清 pending);
       否则 → 入队 AgentQueueApplicationService.in(不打断,当前轮跑完再拉起;子已结束则立即执行)
  停止子:TaskStop(task) → 查 owner=S' → stop_background_agent
       → clearBySessionId(清排队) + stopSubAgentSession(completeMessage(S' 当前 round=END) + 出队 + StopSessionEvent)
       → S' 下次 saveItem 拿 600010001 自停(打断当前轮;S' 仍可被后续 SendMessage 续发)

子 agent 跑完(完成回投,§3.5 基础版已实现):
  [uxa-center] completeMessage(S' round) → 反查 agent_sub_async_agent
       → 取 S' 最终文本,在父 session S 注入一条 user message(锚到父 round R)→ 主下一轮收到
       (被 stop 打断的那一轮属 END 但非完成,需在 §3.5 区分后不投)
```

## 6. 已落地代码索引

**uxa-center(Java)**
- RPC 接口:`api/.../agent/AgentSubAsyncAgentService`(launch / sendMessage / stop / queryWikiSession / queryMeta;**无 close**——2026-06-10 随 `closed` 删除)+ DTO(`LaunchSubAgentRequest{parentSessionId, parentReplyMessageId, subAgentType, framework, mode, prompt, imageUrls, roundExtra, launchToolId}` / `LaunchSubAgentResponse{agentId}` / `SendSubAgentMessageRequest{parentSessionId, targets, message, summary, interrupt}`(`targets`=显式子 id 列表,Python 已展开 `*`;无 parentReplyMessageId——归属只按 session;`interrupt`=true 先软停当前轮再投) / `SendSubAgentMessageResponse{deliveredAgentIds, failedAgentIds}`(按子维度返回谁收到/谁失败,不再只返回 boolean;多目标可部分成功) / `StopSubAgentRequest{agentId}` / `SubAgentMetaRequest{agentId}` → `SubAgentMetaResponse{parentSessionId, subAgentType}` / `SubAgentWikiSessionRequest{subSessionId}` → `SubAgentWikiSessionResponse{userWikiSessionId, orgWikiSessionId}`(据子反查父 + 查 wiki scope 表));impl `service/.../rpc/impl/AgentSubAsyncAgentServiceImpl`。
- 应用层 `application/.../agent/SubAsyncAgentApplicationService`:
  - `launch` → `chatForSubAgent`(钩子内 `insertAssociation` + `copyParentArtifacts`,事务原子,doRun 在事务外);
  - `sendMessage`(按子维度返回 `SendSubAgentMessageResponse{deliveredAgentIds, failedAgentIds}`;逐子 deliver 包 try-catch,单子失败只记 failed、不中断其余子广播)/`resolveTargets`(归属按 parentSessionId、不卡 spawn 轮;targets 由 Python 展开、后端不再有广播分支/closed 过滤)/`deliver`(void,失败抛异常;reply=`replyToolForSubAgent`+`claimPending` 原子认领 / append=`AgentQueueApplicationService.in` 不打断);
  - `stop`(打断当前轮,子可续发) → `AgentQueueApplicationService.clearBySessionId`(清排队 append,防被 drain 重启)+ `stopSubAgentSession` + 残留 `claimPending`;
  - sub→main:`routeAskMainAgentToParent`(markPending + 入队到父,入队 try-catch:失败 `ROUTE_TO_MAIN_FAILED` 回填防悬挂[DF1])、`bindParentRoundIfSubRequest`/`bindParentRound`(由 chat 在 doRun 前同步调[M1];走 `bindParentRoundIfPending` 条件写[DF2])、`failPendingForParentRound(parentSessionId, replyMessageId)`(单轮兜底[M3]);
  - pending 维护:`markPending` / `claimPending`(原子认领,`clearPendingIfMatch` 条件清两字段[H2])。(广播集合无后端维护——展开在 Python,见 §SendMessage;`closed` 已删。)
- `application/.../agent/AgentApplicationCommandService` 新增:`buildChatCmd`(抽取)、`chatForSubAgent`(+`doSubAgentChatInTransaction`)、`replyToolForSubAgent`(+`doReplyToolForSubAgentInTx`,认领-gated)、`stopSubAgentSession`、`softStopRound`(facade `stop` 与 sub 停止共用的软停核心,轮级 key 后缀由调用方传);`chat` 内 doRun 前同步 `bindParentRoundIfSubRequest`(`@Lazy` 注入 SubAsyncAgentApplicationService 破循环)。
- 监听器 `service/.../eventbus/SubAgentCompletionListener`:`ReportSubRoundOnComplete`(CompleteMessageEvent → `reportRoundCompletionToParent`:关联反查 + drain-or-report(自己 `pollNextWithMerge`:弹到 append 起新轮、不投;弹 null 才投,T4)+ 最新轮检查 + `markReportedIfNot` 幂等(T5)+ 末条 model_generation_item 取文 + 排队入父,§3.5);generic `AgentCompleteMessageListener` 对子 session(`isSubSession`)跳过 drain;
- 监听器 `service/.../eventbus/SubAgentAskMainListener`:`RouteAskMainAgent`(SaveItemEvent)/`FailPendingOnParentComplete`(CompleteMessageEvent)/`FailPendingOnParentStop`(StopSessionEvent,共用 `failPendingByMessage`);`SaveItemEvent.isAskMainAgent()`。(绑定不再用 ChatEvent 订阅——见 M1。)
- 数据层:表 `agent_sub_async_agent`(sql:`launch_tool_id` + `UNIQUE(parent_session_id, parent_reply_message_id, launch_tool_id)` 幂等键,`idx_pending_parent`=联合`(parent_session_id, pending_parent_reply_message_id)`;**无 `closed`/`idx_parent_closed`**——2026-06-10 删)、`AgentSubAsyncAgentPO` / `AgentSubAsyncAgentMapper`(`clearPendingIfMatch` / `bindParentRoundIfPending`(条件写[DF2])/ `selectByLaunchToolId` / `listByParentPendingRound` 等 +XML;**无 `listByParentSessionId`**——随 `closed` 删);附件初始拷贝按 rowKey 引用写入子的**不可变基线** `AttachmentDomainService.copyByRowKeyToImmutable`(+`ImmutableAttachmentRepository.saveBatch`;旧 `copyByRowKey`/`AttachmentRepository.saveBatch` 已随之删除)。
- `SessionModeEnums.SUB_AGENT(13, needsHistory=true)`。

**uxarts-agent(Python)**
- 子工具:`tools/agent_message/ask_main_agent_tool.py`(`AskMainAgent`,ToolInterrupted)、`send_message_tool.py`(`SendMessage`)。
- **后端 RPC 接线(已接,2026-06-03)**:`web/dubbo/agent_sub_async_agent_service/`(`service.py` + `types.py`)= `com.uxarts.uxa.center.api.agent.AgentSubAsyncAgentService` 的 Dubbo client(`launch` / `sendMessage` / `stop`),DTO 与 Java 1:1 camelCase。tool 侧 stub 已换成真调用:
  - `tools/subagent/background.py`:`launch_background_agent`(map → `ILaunchSubAgentRequest`,`mode=SUB_AGENT_GENERATE(13)`、`framework` 取父 gctx)、`stop_background_agent`(→ `IStopSubAgentRequest`);(无 `close` client——已随 `closed` 删除);
  - `tools/task/task_tool.py`:TaskStop → 查 owner → `stop_background_agent`。(关不关只走 TaskStop + 由 main 把 task 标 completed/deleted 退出广播;广播范围由 Python 按 pending/in_progress task 过滤,见 S7/§T-T1。)
  - `tools/agent_message/types.py`:`send_agent_message`(→ `ISendSubAgentMessageRequest`,`parentSessionId=from_session_id`;返回 `delivered_agent_ids` / `failed_agent_ids`);
  - 调用点:`tools/subagent/agent_tool.py`(background launch,传 `framework=gctx.framework`)、`tools/task/task_tool.py`(TaskStop)、`send_message_tool.py`(按 delivered/failed 渲染结果)。失败抛 `RpcException`,由各 tool 既有 try/except 降级。
- 父侧识别子汇报:`task/prototype/handlers/sub_agent_report.py`(`@register_business_handler("sub_agent_report")`,读 `fromSubAgent`/`subAgentReportSuccess` 注入 `<sub_agent_report>` 提示对账 task/续发/接手)。
- 父侧识别子请求:`task/prototype/handlers/sub_agent_request.py`(`@register_business_handler("sub_agent_request")`,读 `businessParams.fromSubAgent` 注入 `<sub_agent_request>` system-reminder)。
- 子 session 输入/初始化:`task/sub_designer/sub_designer_user_input_creator.py`、`sub_designer_agent_init_message.py`;mode13 分发 `task/getui_generate_task.py`。
- prompt:`prompts/superun/background_agent/main_workflow.txt`(主侧:启动/收发/处理子请求)、`sub_workflow.txt`(子侧)。

## 7. 待 review / 待定

> 决定与待定的**权威清单**见 `background_agent_open_questions.md`(已逐条记决定)。本节只留后端实现前仍需落的几条:

- **#2 委托 PENDING 完整生命周期**(超时/父 stop/失败的级联与 deadline 巡检)——"后面做";单轮兜底(主那轮没回 → 自动失败,见 §3.3)已定。
- **#4 完成回投锚定分支**(preMessageId 锚到父分支 tip + active 校验 + 幂等;stale 写 inbox 不自动 run)——"后面做"。
- **#6 打断双轮并发 token**——"后面做"。
- 清理策略:父 session 结束 / 删除 / 分支回退时,级联 stop 关联的未结束 sub-agent(挂 `StopSessionEvent` 监听)。
- 其余(独立 session、`agent_id=sub_session_id`、`sub-designer` 默认类型、TaskStop/TaskList 复用、sub→main 排队、SendMessage 自动 reply/append)均**已定**,见 open_questions。
