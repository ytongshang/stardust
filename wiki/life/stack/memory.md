---
title: Memory
type: reference
created: 2026-05-28
updated: 2026-05-30
tags:
  - life
  - stack
  - memory
  - yatoro
  - agent-memory
sources:
  - "Slock task #8 #yatoro:aca60753 — memory system survey (2026-05-28)"
  - "Local clone /Users/rancune/Work/rancune/work/github-open-source/PilotDeck @ c8a4606"
  - "Local clone /Users/rancune/Work/rancune/work/github-open-source/ClawXMemory @ bcd66c5"
  - "Local clone /Users/rancune/Work/rancune/work/github-open-source/openclaw @ 79e733cc"
  - "Local clone /Users/rancune/Work/rancune/work/github-open-source/codex @ a4ed6c5"
  - "Local clone /Users/rancune/Work/rancune/work/github-open-source/claude-code @ 3bb4455"
  - "Local clone /Users/rancune/Work/rancune/work/github-open-source/mempalace @ 6957c7e"
  - "Local clone /Users/rancune/Work/rancune/work/github-open-source/mem0 @ 9328c36"
  - "Local clone /Users/rancune/Work/rancune/work/github-open-source/hermes-agent @ a30480bd2"
  - "Slock thread #yatoro:aca60753 msg=423b27c4 — request for detailed storage/scope/evolution breakdown (2026-05-29)"
  - "Slock thread #yatoro:aca60753 msg=7851f932 — focus on open-source implementations, not Yatoro implementation design (2026-05-29)"
  - "Slock task #10 #yatoro:f2ac03a2 — analyze NousResearch/hermes-agent memory implementation (2026-05-29)"
  - "Slock task #11 #yatoro:60f8bbf7 — Yatoro memory pipeline executable design (2026-05-30)"
---

# Memory

> 开源 memory 系统学习笔记。本文暂不设计 Yatoro 实现，重点看这些项目做了什么、怎么存、按什么维度分层、怎么从低层演化到高层、以及 Dream / AutoDream / consolidation 如何整理记忆。

Yatoro 自身的可执行 memory pipeline 设计见 [[life/stack/memory-pipeline|Memory Pipeline]]。

## 阅读结论

这几类系统大致分成三条路线：

- **Markdown-first**：ClawXMemory。SQLite 只做 raw session queue / pipeline state，长期语义真源是 Markdown files，Dream 会重写和归并文件。
- **DB -> file workspace -> consolidation**：Codex。先把 session/rollout 抽成 stage-1 DB record，再用全局 lock 和 git-backed workspace 让 consolidation agent 更新 `MEMORY.md`、summary、skills。
- **Vector-first retrieval**：MemPalace、mem0。语义真源主要在 vector/Chroma store；文件只做 identity、配置或外部说明。它们更重 retrieval 和 scope/filter。

OpenClaw 更像 workspace memory runtime：`MEMORY.md` + `memory/*.md` + memory tools，不强制自动整理。Claude Code public repo 只暴露文件、settings、hooks、plugins 这些上下文通道，没开放 memory backend。

Hermes Agent 是组合式 memory runtime：内置 `MEMORY.md` / `USER.md` 是小而强的 always-on curated memory；`state.db` 保存完整 session history 并提供 FTS5 `session_search`；外部 MemoryProvider 插件负责更深的 semantic / graph / profile / knowledge-base recall。

## 分层

这些开源实现虽然形态不同，但大多都有一条从低到高的链路：

| 层                             | 生命周期                         | 常见存储                                                      | 典型项目                                 | 作用                             |
| ----------------------------- | ---------------------------- | --------------------------------------------------------- | ------------------------------------ | ------------------------------ |
| Raw conversation / transcript | 当前会话或一次 rollout              | runtime state、JSONL、hook 输入、session note                  | Codex、ClawXMemory、MemPalace、OpenClaw | 保留原始证据，不直接当长期知识                |
| Queue / extraction input      | 待处理批次                        | SQLite row、state DB job、transcript file list              | Codex、ClawXMemory、mem0               | 让后台 worker 可 claim、retry、跳过、限流 |
| Candidate memory              | 单次会话抽出的候选                    | DB stage-1 output、Markdown fragment、vector memory payload | Codex、ClawXMemory、mem0               | 把原始对话压成可复用事实、偏好、决策、摘要          |
| Scoped long-term memory       | 用户/项目/agent/run/topic 下的长期知识 | Markdown file-memory、vector collection、Chroma drawer      | ClawXMemory、mem0、MemPalace           | 按 scope 组织，避免跨人、跨项目、跨任务污染      |
| Retrieval / index layer       | 按需召回                         | manifest、BM25/embedding index、closet pointer、entity store | OpenClaw、MemPalace、mem0、ClawXMemory  | 控制上下文注入量，只取与当前问题相关的部分          |
| Consolidated artifact         | 更高层稳定产物                      | `MEMORY.md`、summary、skills、project files、profile files    | Codex、ClawXMemory                    | 把候选提升、归并、删除、改写成可长期维护的结构        |

## Markdown 文件 vs SQLite / DB

开源项目里常见的分工是：

| 组件 | 适合做什么 | 不适合做什么 |
|---|---|---|
| Markdown / text files | 人和 Agent 都能读写的长期真源；项目状态、偏好、决策、SOP、可审计记忆 | 高频队列、锁、并发 claim、快速筛选、复杂 trace |
| SQLite / state DB | raw session 队列、indexed 标记、pipeline state、lease/lock、trace、history、usage_count、last_usage | 长期语义真源；如果只在 DB 里，后续人和 Agent 很难维护 |
| Vector DB / Chroma / embedding store | 大规模语义搜索、相似度召回、entity boost、wing/room metadata filter | 规范化的工作流规则和最终决策；需要额外治理防止污染 |
| Git-backed workspace | consolidation 前后的 diff、可审计自动改写、可回滚 baseline | 高频 transient state |

所以 “Markdown 文件和 SQLite 分别做什么” 的答案通常是：**Markdown 存最终知识，SQLite 存流水线状态**。例外是 MemPalace/mem0：它们本身是检索/记忆产品，vector store 就是主要语义存储；Markdown 只做 identity 或外部文档。

## 调研对比

| 项目 | 存储真源 | 主要 scope | 写入方式 | 读取方式 | 进化/整理 |
|---|---|---|---|---|---|
| PilotDeck + ClawXMemory | ClawXMemory 的 Markdown file-memory；SQLite 只做 raw session/control plane | global user、project、feedback、session | `captureTurn` 写 L0 raw sessions，后台 heartbeat 提取到 file-memory | answer-time hook 单项目检索，注入 `<memory-context>` | dream review 重写项目/用户文件，提升 `_tmp`，删除 absorbed/superseded |
| OpenClaw | workspace `MEMORY.md` + `memory/*.md`，可接 builtin/qmd backend | workspace、session、tool backend | hooks 把最近消息落到 `memory/YYYY-MM-DD-HHMM.md`；root `MEMORY.md` 人工维护 | 有 memory tools 时只注入指针，让模型按需 `memory_search` / `memory_get` | 依赖 workspace 文件维护；子 agent 不自动继承 parent memory |
| Codex | state DB stage-1 + `~/.codex/memories` git-backed workspace | rollout/thread、global memory workspace | startup 后台抽取 recent rollout -> DB stage-1 | read path 注入 developer instruction + citation/usage telemetry | Phase 2 串行 consolidation agent 基于 workspace diff 更新 `MEMORY.md`、summary、skills |
| MemPalace | ChromaDB palace；identity 是 `~/.mempalace/identity.txt` | identity、wing、room、drawer | mine project/convo files，precompact hook 同步保存 transcript | L0/L1 wake-up + L2 wing/room recall + L3 semantic search | layer scoring、idempotent re-mine、closet pointer、graph/tunnel；偏检索系统 |
| mem0 | vector store memory + SQLite history/messages | user_id、agent_id、run_id、metadata | `add` 用 LLM 从 messages 提取，或 `infer=false` 存 raw；必须带 scope | `search/get_all` 用 filters；支持 entity store boost | update/delete/history；session messages 只保留最近 10 条作 extraction context |
| Hermes Agent | `memories/MEMORY.md` + `USER.md`；`state.db`；可选外部 provider | profile/HERMES_HOME、memory/user、session、user_id、agent_id、provider-specific bank/container/project | `memory` tool 写文件；session 自动进 `state.db`；provider `sync_turn` / hooks 异步写外部 backend | frozen prompt snapshot、`session_search`、provider prefetch 注入 `<memory-context>`、provider tools | 后台 memory/skill review；provider session_end/pre_compress；没有统一 Dream 重写长期文件 |
| Claude Code public repo | 公开 repo 未暴露核心 memory backend | project/user settings、hooks、plugins | 通过 `CLAUDE.md`、settings、hooks、plugins 间接注入/保存 | session resume、hooks、plugins 提供上下文通道 | 不能从公开代码判断其闭源 memory 内部实现 |

## 逐项目解剖

### PilotDeck + ClawXMemory

PilotDeck 自己不是长期 memory store，它是把 Agent turn 接到 EdgeClaw/ClawXMemory 的适配层。

PilotDeck 的接口层：

```text
MemoryResolver
├── retrieve(query, sessionId, projectRoot, recentMessages)
└── captureTurn(sessionId, projectRoot, messages, errored)
```

PilotDeck 做的事：

- `retrieve`：把当前 query、recent messages、`workspaceHint = projectRoot` 传给 `EdgeClawMemoryService.retrieveContext`，拿回 `systemContext` 后注入 prompt。
- `captureTurn`：把当前 turn 规范化成 `{ role, content, msgId }`，以 `sessionKey = sessionId` 写入底层 memory service。
- config 决定是否启用 memory、provider、`rootDir`、`captureStrategy`、是否包含 assistant、`maxMessageChars`、`heartbeatBatchSize`、`autoIndexIntervalMinutes`、`autoDreamIntervalMinutes`。
- provider 是 per-project scope：默认按 `projectRoot` 组织，设置 `rootDir` 时可把 memory root 固定到其他位置。

ClawXMemory 的长期结构：

```text
<memory-root>/
├── global/
│   ├── User/
│   │   └── user-profile.md
│   └── MEMORY.md                  # derived manifest, not semantic truth
└── projects/
    ├── _tmp/
    │   ├── Project/*.md            # 无法确定正式 project 时先放这里
    │   ├── Feedback/*.md
    │   └── MEMORY.md               # derived manifest
    └── project_<hash>/
        ├── project.meta.md         # 正式项目 identity / aliases / status
        ├── Project/*.md            # 项目状态、决策、下一步、blocker、timeline
        ├── Feedback/*.md           # 用户协作偏好、交付偏好、反馈规则
        └── MEMORY.md               # derived manifest
```

Markdown frontmatter 的关键字段：

```yaml
name: <memory item name>
description: <short summary>
type: user | project | feedback
scope: global | project
project_id: project_<hash> | _tmp
updated_at: <iso time>
captured_at: <iso time>
source_session_key: <session key>
deprecated: true | false
dream_attempts: <number>
```

SQLite 的职责：

```sql
l0_sessions(
  l0_index_id,
  session_key,
  timestamp,
  messages_json,
  source,
  indexed,
  created_at
)

pipeline_state(
  state_key,
  state_json,
  updated_at
)
```

分工：

- `l0_sessions` 是原始 session 队列。`indexed = 0` 表示还没被 background indexing 消费。
- `pipeline_state` 放 runtime 设置和 trace，例如 `indexingSettings`、`lastIndexedAt`、`lastDreamAt`、recent case/index/dream traces。
- SQLite 不作为长期记忆真源；长期语义都在 Markdown files。

从低等级到高等级：

1. **Raw session**：PilotDeck/OpenClaw hook 把 turn 写成 `l0_sessions`。
2. **Index candidate**：heartbeat 批量读取未 indexed session，让 LLM 抽取 `MemoryCandidate`。
3. **File memory**：candidate 写入 `user-profile.md`、`Project/*.md`、`Feedback/*.md`；不能确定项目时写入 `_tmp`。
4. **Recall manifest**：`MEMORY.md` manifest 是从文件生成的目录，用于 UI/debug，不是召回真源。
5. **Answer-time recall**：运行时先判断 `none / user / project_memory`；如果是 project memory，只选一个 project，读 `project.meta.md`，扫描 `Project/Feedback` 文件头，再加载 top-k 文件正文。
6. **Dream**：读 formal + `_tmp` file-memory，规划 project 合并/改名/提升，重写 project files/user profile，删除 absorbed/superseded 文件，记录 dream trace。

这个设计的核心是：**session 原文可以排队，长期知识必须变成可读可改的 Markdown。**

### OpenClaw

OpenClaw 的 memory 更像 workspace bootstrap + on-demand retrieval，不是 ClawXMemory 那种自动长期整理系统。

主要文件：

```text
<workspace>/
├── MEMORY.md              # canonical root memory file
├── memory.md              # legacy root memory filename
└── memory/
    └── YYYY-MM-DD-HHMM.md # session-memory hook 生成的日常记忆片段
```

维度：

- **workspace**：`MEMORY.md` 属于当前 workspace。
- **session/daily file**：`session-memory` hook 在 `/new`、`/reset` 时把最近 15 条 user/assistant message 保存到 `memory/YYYY-MM-DD-HHMM.md`。
- **backend**：memory backend 可是 `builtin` 或 `qmd`；qmd 可配置 index paths、session exportDir、retentionDays、update interval、search limits。
- **channel**：Discord 文档里说明 DM 默认加载 long-term memory，guild channel 不自动加载，避免公共频道泄露或过度注入。
- **sub-agent**：子 agent 只注入 `AGENTS.md` 和 `TOOLS.md`，不自动继承 parent 的 `MEMORY.md`、`USER.md`、`IDENTITY.md`。

Prompt 读取方式：

- `MEMORY.md` 存在时可作为 workspace bootstrap 资源。
- 对 native Codex turns，如果 memory tools 可用，OpenClaw 不把完整 `MEMORY.md` 每轮塞进 prompt，而是注入一个 workspace-memory 指针，让模型按需调用 `memory_search` / `memory_get`。
- 如果 memory tools 不可用，才走 bounded turn-context fallback。

从低等级到高等级：

1. **Session raw-ish note**：hook 把最近消息保存到 `memory/*.md`。
2. **Searchable memory corpus**：builtin/qmd 对 `MEMORY.md`、`memory/*.md` 或配置 paths 建索引。
3. **On-demand recall**：模型根据问题调用 `memory_search` / `memory_get`。
4. **Manual curation**：人或 agent 把稳定内容整理进 `MEMORY.md` 或其他 workspace docs。

OpenClaw 没有强制的 AutoDream。它的核心取舍是：**root `MEMORY.md` 保持短，日常碎片放 `memory/*.md`，用工具按需检索。**

### Codex

Codex 的 memory 是典型两阶段 pipeline：先从 session/rollout 里抽候选，再让受限 consolidation agent 改全局 memory workspace。

低层输入：

- rollout / thread state 在 state DB 里。
- Phase 1 只选 recent、interactive、idle enough、未被其他 worker claim、在 age/scan/claim limit 内的 rollout。
- 每个 job 都有 DB claim/lease，避免并发重复处理。

Phase 1 输出：

```json
{
  "raw_memory": "detailed markdown for one rollout",
  "rollout_summary": "compact routing/index line",
  "rollout_slug": "optional filename slug"
}
```

这些 stage-1 outputs 存在 state DB 中，用于后续 selection、usage tracking、retry/backoff。

Phase 2 workspace：

```text
~/.codex/memories/
├── .git/                       # git baseline, used for workspace diff
├── raw_memories.md             # selected stage-1 raw memories merged by stable thread order
├── rollout_summaries/
│   └── <timestamp>-<hash>-<slug>.md
├── phase2_workspace_diff.md    # current workspace diff for consolidation agent
├── MEMORY.md                   # higher-level consolidated output
├── memory_summary.md           # higher-level consolidated output
└── skills/                     # possible generated/refined skills
```

维度：

- **rollout/thread**：Phase 1 是 per-thread/per-rollout extraction。
- **cwd/project**：stage-1 record 带 cwd、rollout path、git branch 等上下文。
- **global memory workspace**：Phase 2 是全局合并，不再按单 thread 回答，而是更新长期文件。
- **usage/recentness**：Phase 2 selection 按 `usage_count`、`last_usage`、`generated_at`、`max_unused_days` 选 bounded top-N。

从低等级到高等级：

1. **Rollout transcript**：原始交互存在 rollout/state。
2. **Stage-1 raw memory**：后台抽取一个 rollout 的详细 memory 和摘要。
3. **Selected phase-2 inputs**：按使用次数和新鲜度选择 stage-1 outputs。
4. **Memory workspace diff**：把候选同步成 `raw_memories.md` 和 `rollout_summaries/`，用 git diff 判断是否有变化。
5. **Consolidated memory**：受限 sub-agent 读取 diff，更新 `MEMORY.md`、summary、skills 等长期产物。
6. **Baseline reset**：成功后重置 git baseline，下一轮只看新 diff。

DB 和文件分工：

- DB：claim/lease/retry、stage-1 outputs、selection、usage、watermark。
- 文件 workspace：agent 可读写的长期 memory 真源和 consolidation diff。

这个设计的关键启发是：**自动抽候选可以在 DB，最终改长期文件要通过可审 diff。**

### MemPalace

MemPalace 是检索型 memory palace，和 ClawXMemory 的 Markdown-first 不同：主要真源在 ChromaDB collection。

存储结构：

```text
~/.mempalace/
├── identity.txt              # L0 identity, plain text
├── config.json
└── <palace dir>/
    ├── chroma.sqlite3
    ├── mempalace_drawers     # Chroma collection: verbatim chunks
    └── mempalace_closets     # Chroma collection: compact pointer/index layer
```

drawer metadata 典型字段：

```json
{
  "wing": "wing_api or project/person/topic",
  "room": "technical / architecture / decisions / ...",
  "hall": "hall_facts / hall_events / hall_preferences / ...",
  "source_file": "/path/to/transcript-or-file",
  "chunk_index": 0,
  "added_by": "mempalace",
  "filed_at": "iso time",
  "ingest_mode": "convos | projects",
  "extract_mode": "exchange | general",
  "normalize_version": 2
}
```

维度：

- **L0 identity**：`identity.txt`，总是加载。
- **L1 essential story**：从 palace 里最高 importance / emotional weight 的 drawers 生成，bounded 到约 3200 chars。
- **L2 recall**：按 wing / room filter 取 drawers。
- **L3 search**：全 palace semantic search。
- **wing**：人、项目、topic 的顶层分区。
- **room**：一个 wing 内的具体主题。
- **hall**：事实、事件、发现、偏好、建议等概念分类。
- **drawer**：原文 chunk，MemPalace 强调 verbatim，而不是只存 summary。
- **closet**：紧凑 pointer，指回 drawer ids，作为索引/导航层。

从低等级到高等级：

1. **Source file / transcript**：project files、Claude/Codex/Gemini transcript、ChatGPT/Slack/plain text。
2. **Normalize**：清理 Claude Code JSONL/system tags/hook chrome 等噪声。
3. **Chunk**：conversation 默认按 Q+A exchange chunk；general mode 会抽 decisions/preferences/milestones/problems/emotions。
4. **Route**：根据 path、keywords、内容检测 wing/room/hall。
5. **Drawer upsert**：写入 `mempalace_drawers`，按 source_file + extract_mode + chunk_index 做稳定 id，重复 mine 不重复。
6. **Closet build**：为 source file 构造 topic/entity/date-line pointer，写入 `mempalace_closets`。
7. **Wake-up / recall / search**：L0/L1 进启动上下文；L2/L3 按需检索。

Auto-save / compaction：

- `mempal_precompact_hook.sh` 在 Claude Code / Codex PreCompact 前同步运行。
- 它读取 hook 输入里的 `session_id` 和 `transcript_path`，对 transcript parent dir 执行 `mempalace mine --mode convos`。
- 可选 `MEMPAL_DIR` 额外 mine project files。

这里没有 ClawXMemory 那种 “Dream 重写 Markdown 项目文件” 的主路径；它更像是：**持续把原文 filing 到可检索宫殿，用层级和索引控制召回量。**

### mem0

mem0 是 API/SDK 风格的 memory store，核心不是文件，而是 scoped vector memories。

主要存储：

```text
Vector store main collection
├── id
├── embedding
└── payload:
    ├── data                # memory text
    ├── hash                # dedupe
    ├── text_lemmatized     # keyword/BM25 support
    ├── created_at
    ├── updated_at
    ├── user_id
    ├── agent_id
    ├── run_id
    ├── actor_id
    ├── role
    ├── memory_type         # e.g. procedural_memory
    └── metadata...

Vector store entity collection: <collection_name>_entities
├── entity text / type
└── linked_memory_ids

SQLite
├── history(memory_id, old_memory, new_memory, event, timestamps, actor_id, role)
└── messages(session_scope, role, content, name, created_at)
```

维度：

- **user_id**：跟随人的长期偏好/事实。
- **agent_id**：跟随 agent persona、工作方式、procedural memory。
- **run_id**：短期 flow / ticket / session。
- **metadata**：业务标签、分类、时间等。
- **actor_id / role**：多参与者消息归属。

mem0 强制至少提供 `user_id`、`agent_id`、`run_id` 之一；search/get_all/delete 也要求 filters 带 scope。Platform docs 还强调 user/agent/app/run 的实体隔离，避免不同用户或 agent 的 memory 混在一起。

从低等级到高等级：

1. **Messages**：用户传入 string 或 messages。
2. **Recent context**：SQLite `messages` 表取同一 `session_scope` 最近 10 条，给 extraction 用。
3. **Existing memory retrieval**：按 scope filters 检索已有 memories。
4. **LLM extraction**：LLM 从新消息 + 已有 memory + recent messages 里抽取 key facts。
5. **Dedup**：按 hash 去重。
6. **Vector insert/update**：写入主 vector store。
7. **History**：SQLite `history` 记录 ADD/UPDATE/DELETE。
8. **Entity link**：抽实体，写入 entity store，search 时给相关 memory boost。
9. **Message save**：保存最近消息，且每个 session_scope 只保留最近 10 条。

Markdown vs SQLite：

- mem0 没有 Markdown 长期真源。
- vector store payload 是语义真源。
- SQLite 是 audit history + recent-message context，不是长期 memory 本体。

mem0 的启发是：**如果要自动记忆，scope 必须强制显式；如果 scope 不清楚，就不要写。**

### Hermes Agent

Hermes Agent 的 memory 实现不是一个单独 backend，而是四个层叠系统：

```text
Hermes runtime
├── built-in curated memory
│   └── $HERMES_HOME/memories/
│       ├── MEMORY.md       # agent personal notes
│       └── USER.md         # user profile
├── session recall
│   └── $HERMES_HOME/state.db
│       ├── sessions
│       ├── messages
│       ├── messages_fts
│       └── messages_fts_trigram
├── memory provider manager
│   ├── agent/memory_provider.py
│   └── agent/memory_manager.py
└── provider plugins
    ├── plugins/memory/honcho
    ├── plugins/memory/mem0
    ├── plugins/memory/supermemory
    ├── plugins/memory/openviking
    ├── plugins/memory/hindsight
    ├── plugins/memory/holographic
    ├── plugins/memory/retaindb
    └── plugins/memory/byterover
```

内置 `MEMORY.md` / `USER.md`：

- 存储位置：`$HERMES_HOME/memories/MEMORY.md` 和 `$HERMES_HOME/memories/USER.md`。
- `MEMORY.md` 存 agent 自己的长期笔记：环境事实、项目约定、工具 quirks、可复用经验。
- `USER.md` 存用户画像：偏好、沟通风格、工作方式、长期期望。
- 两个文件都是 `§` delimiter 分隔的 entry list，不是普通自由 Markdown。
- 默认容量很小：`MEMORY.md` 约 2200 chars，`USER.md` 约 1375 chars。
- 启动时 `MemoryStore.load_from_disk()` 读文件并生成 frozen snapshot；中途 `memory` tool 写盘，但 system prompt 里的 snapshot 不变，下一 session 才刷新。
- 写入支持 `add` / `replace` / `remove`，`replace` 和 `remove` 用短 substring 匹配条目。
- 写入前会做 threat-pattern scan；读入 system prompt 前也会 scan，命中时 entry 在 prompt snapshot 里替换为 `[BLOCKED: ...]`，但 live state 保留原文方便用户删除。
- 写文件用 lock file + atomic temp rename，防止并发写和半写文件；如果发现外部 writer 让文件不能 round-trip，会先备份 `.bak.<ts>` 并拒绝本次 mutation。

`state.db` session recall：

- 存储位置：`$HERMES_HOME/state.db`。
- `sessions` 表保存 session metadata：source、user_id、model、system_prompt、parent_session_id、token/cost counters、title、handoff state 等。
- `messages` 表保存完整 message history：role、content、tool calls、tool name、reasoning、platform_message_id、observed 等。
- `messages_fts` 是 FTS5 full-text index；`messages_fts_trigram` 是 trigram FTS5 index，专门改善 CJK / substring search。
- 这部分不是“长期语义真源”，而是历史证据和检索层。`session_search` 没有 LLM summarization，直接返回 DB 里的真实 messages。
- `session_search` 有三种调用形态：query discovery、session+message scroll、无参 browse。

MemoryProvider 插件层：

- 抽象接口在 `agent/memory_provider.py`，统一生命周期：`initialize`、`system_prompt_block`、`prefetch`、`queue_prefetch`、`sync_turn`、`on_session_end`、`on_pre_compress`、`on_memory_write`、provider tools、`shutdown`。
- 调度器在 `agent/memory_manager.py`。内置 memory 之外，**同一时间只允许一个 external provider**，避免 tool schema 膨胀和 backend 冲突。
- `system_prompt_block()` 只放静态 provider 状态；动态 recall 由 `prefetch()` 返回，在当前 turn 的 user message 后追加 `<memory-context>` block，不写回原始 session messages。
- 完成一轮后，`sync_all()` 把 `(user_content, assistant_content)` 发给 provider；`queue_prefetch_all()` 为下一轮预取。
- 如果模型调用内置 `memory` tool 写 `MEMORY.md` / `USER.md`，Hermes 会通过 `on_memory_write()` bridge 把这次写入镜像给外部 provider。
- session 结束和 context compression 前分别触发 `on_session_end()` / `on_pre_compress()`，给 provider 最后一轮抽取或 flush 的机会。

外部 provider 的 scope / 存储差异：

| Provider | 主要存储 | Scope 维度 | 写入 | 读取 |
|---|---|---|---|---|
| Honcho | Honcho Cloud/self-hosted peers + sessions + conclusions | workspace、user peer、AI peer、session key、profile | 每 turn 写 user/assistant messages；`honcho_conclude` 存 conclusion；可迁移 `MEMORY.md/USER.md/SOUL.md` | base context（session summary / user representation / peer cards）+ dialectic reasoning + tools |
| mem0 | mem0 Platform vector memory | `user_id`、`agent_id` | `sync_turn` 让 mem0 server-side extraction；`mem0_conclude` 用 `infer=false` 存 verbatim fact | `mem0_profile` / `mem0_search`，prefetch 默认按 `user_id` search |
| Supermemory | Supermemory documents/memories under container tag | container tag、profile identity、optional custom containers | cleaned conversation turn、session ingest、explicit store | profile static/dynamic facts + search results，支持 hybrid/memories/documents search mode |
| OpenViking | OpenViking context DB + `viking://` filesystem | account、user、agent、session、URI | session messages -> commit 后提取 profile/preferences/entities/events/cases/patterns；memory writes 写 `viking://user/<user>/memories/...` | `viking_search`、`viking_read` abstract/overview/full、browse |
| Hindsight | Hindsight bank/document API | bank id / template、profile、workspace、platform、user、session tags | turns 组成 document batch，经 writer queue retain；支持 cloud/local embedded/local external | recall / reflect；默认 recall observation 类型 |
| Holographic | local SQLite `memory_store.db` | fact category、entity、trust score、HRR bank | explicit `fact_store(add)`；可选 session-end auto_extract；mirror built-in memory writes | FTS5 + Jaccard + trust + optional HRR compositional retrieval |
| RetainDB | RetainDB Cloud + local `retaindb_queue.db` | project、user_id、session_id、agent_id | SQLite-backed write-behind queue，crash-safe replay；explicit remember；file ingest | profile、semantic search、context overlay、dialectic user synthesis、agent self-model |
| ByteRover | external `brv` CLI knowledge tree | ByteRover workspace cwd / cloud identity | `brv curate` turn summaries、pre-compression flush、memory mirror | synchronous `brv query` before model call |

从低等级到高等级：

1. **Raw messages**：每个 session 的原始 messages 进 `state.db.messages`，FTS5 自动索引。
2. **Always-on snapshot**：启动时把 `MEMORY.md` / `USER.md` 渲染成 frozen prompt block。
3. **Per-turn recall**：external provider 在 turn start 前 `prefetch()`，返回动态 recall，Hermes 用 `<memory-context>` fence 注入当前 user message。
4. **Post-turn capture**：turn 完成后 `sync_turn()` 异步发给 provider，provider 可抽 fact、建 profile、写 vector store、写 session DB。
5. **Explicit memory**：模型用 `memory` tool 写内置文件；同时通过 bridge 通知 external provider。
6. **Session boundary extraction**：session end / pre-compress hook 触发 provider 自己的 flush、commit、retain、ingest。
7. **Background review**：每隔 `memory.nudge_interval`，Hermes fork 一个 review agent 检查对话是否有值得写入内置 memory 的稳定偏好或用户信息；这个 fork `skip_memory=True`，不会污染 external provider。

Markdown vs SQLite / DB：

- `MEMORY.md` / `USER.md` 是手工或 agent-curated 的小型长期真源，适合 always-on facts，不适合任务日志。
- `state.db` 是完整历史和可搜索证据，适合找过去讨论，不适合直接变成 prompt 常驻 memory。
- provider DB/API 是深层召回层：有的本地 SQLite（Holographic、RetainDB queue），有的云端 vector/graph/profile（Honcho、mem0、Supermemory、OpenViking、Hindsight、RetainDB）。
- Hermes 的核心取舍是：**小文件常驻 + SQLite 全量历史 + 一个可插拔深层 provider**。它不像 ClawXMemory 那样把所有长期语义都整理成项目 Markdown，也不像 mem0 那样完全 vector-first。

Dream / AutoDream：

- Hermes 没有统一的 ClawXMemory-style Dream：不会定期重写所有长期 Markdown，也没有 project `_tmp` promote / absorbed delete 的统一流程。
- 最接近 AutoDream 的是两处：后台 memory/skill review（从对话中挑稳定内容写 memory/skill）和 external provider 的 session-end / pre-compress extraction。
- 但这些都是“局部提升”：内置 memory 是小条目更新；skills 是 skill library 更新；provider 各自做自己的 profile/vector/graph consolidation。
- 保护措施主要是：外部 provider best-effort 不能阻塞主回答；动态 recall 用 `<memory-context>` fence；streaming scrubber 防止 memory-context 泄漏到 UI；background review 禁止触碰 external provider。

### Claude Code public repo

`anthropics/claude-code` 公开 repo 主要是 docs、plugins、examples，不包含闭源 Claude Code runtime 的 memory backend。

能确认的公开结构：

- `CLAUDE.md` / project instructions / user settings 提供长期上下文入口。
- hooks 可以在 SessionStart、PreCompact、PostCompact 等生命周期加载或保存内容。
- plugins 可以携带 agents、commands、skills、hooks。
- changelog 提到 session resume、background sessions、session metadata、preCompact/postCompact 等能力。

不能从这个 repo 证明：

- 是否有内置长期 memory DB。
- 是否有类似 Codex Phase 1/Phase 2 的后台 consolidation。
- memory 是按 project、user、session 还是 global 存储。

所以它只能作为 **文件、settings、hooks、plugins 如何形成上下文通道** 的参考；不要把它当作可研究的开源 memory backend。

## 关键模式

### ClawXMemory：Markdown 是长期真源

- SQLite 存 `l0_sessions` 和 `pipeline_state`，不是长期语义真源。
- 长期记忆是文件：`global/User/user-profile.md`、`projects/<projectId>/project.meta.md`、`Project/*.md`、`Feedback/*.md`、`projects/_tmp/*`。
- 检索时先判断 `none / user / project_memory`，项目记忆一次只选一个 project，避免跨项目混杂。
- Dream 会读 formal + `_tmp` file-memory，重写项目文件、提升 `_tmp`、删除已吸收文件、更新 user profile。

适用场景：需要人和 Agent 共同维护长期知识，并且希望每次改写都可读、可审、可 diff。

### Codex：先抽取，再全局合并

- Phase 1 从 recent rollout 抽取 `raw_memory`、`rollout_summary`、`rollout_slug`，存 DB。
- Phase 2 拿全局 lock，把 stage-1 输出同步到 `~/.codex/memories`，生成 `raw_memories.md`、`rollout_summaries/`、`phase2_workspace_diff.md`。
- 如果 workspace 有变化，启动一个受限 consolidation sub-agent：无网络、无需 approval、只能写本地 memory workspace。
- Consolidation agent 负责更新更高层的 `MEMORY.md`、`memory_summary.md`、`skills/`。

适用场景：希望后台自动抽候选，但把长期文件改写放在受控、可 diff、可回滚的 consolidation 阶段。

### MemPalace：检索型 memory palace

- L0 identity 永久加载，L1 从高权重 drawer 自动生成 wake-up，L2 按 wing/room recall，L3 做 full semantic search。
- Wing 是人/项目/topic，room 是具体主题，drawer 保存原文 chunk，closet 是指向 drawer 的紧凑索引。
- PreCompact hook 在上下文压缩前同步 mine transcript，避免 compaction 前丢细节。

这个模式适合“大量历史会话检索”，但它的整理方式不是 Markdown project rewrite，而是通过结构化检索层控制召回。

### mem0：scope 必须显式

- 写入和读取都围绕 `user_id`、`agent_id`、`run_id`，并可附加 metadata。
- `run_id` 适合短期 session/ticket；`user_id` 适合长期个人偏好；`agent_id` 适合 agent 自身风格或 procedural memory。
- `history` 记录 ADD/UPDATE/DELETE；`messages` 表保存当前 scope 最近 10 条消息，用于下次 extraction context。

这个模式提醒：任何自动 memory 都必须带明确 scope，否则很快会污染检索结果。

### Hermes Agent：小文件常驻 + session DB + provider 插件

- `MEMORY.md` / `USER.md` 是极小的 always-on curated memory，启动时 frozen snapshot 进 system prompt。
- `state.db` 是完整 session history + FTS5 / trigram search，负责“过去聊过什么”的证据检索，不是语义真源。
- external provider 只允许一个，负责深层 recall / semantic search / user modeling；动态 recall 用 `<memory-context>` 注入当前 turn。
- 后台 review agent 会定期判断是否需要写内置 memory / skill，但它禁用 external provider，避免把 review harness 本身写进远端 memory。

适用场景：既想保留小而稳定的 prompt memory，又想拥有完整历史搜索和可替换的深层 memory backend。

## Dream / AutoDream / Consolidation 对比

| 项目 | 有没有 Dream/AutoDream | 它整理什么 | 它怎么改写 | 关键保护 |
|---|---|---|---|---|
| ClawXMemory | 有明确 Dream | formal project files、`_tmp` files、user profile | 全局 project plan -> per-project rewrite -> promote `_tmp` -> delete absorbed/superseded -> rewrite user profile | Markdown 真源、trace、`dream_attempts`、不新增独立 summary layer |
| Codex | 没叫 Dream，但 Phase 2 是类似 consolidation | stage-1 raw memories、rollout summaries、memory workspace diff | 全局 lock -> sync files -> git diff -> restricted consolidation agent 更新长期 artifacts | DB claim/lease、git baseline、无网络/无 approval/local write only |
| MemPalace | 没有 project rewrite 型 Dream | Chroma drawers / closets / graph/tunnels | re-mine 时按 normalize_version 清旧 drawer，L1 每次从高权重 drawer 生成 | 稳定 drawer id、source_file 去重、PreCompact 同步保存 |
| mem0 | 没有全局 Dream | 单条 memory 和 entity links | add/extract、update、delete、history、entity boost | 强制 scope filters、hash dedupe、history audit |
| OpenClaw | 没有强制 AutoDream | workspace `MEMORY.md`、`memory/*.md` | hook 写 session note，人工/agent 按需整理 root memory | memory tools on-demand，不把全部 memory 注入 prompt |
| Hermes Agent | 没有统一 Dream；有后台 review + provider hooks | `MEMORY.md` / `USER.md`、skills、external provider backend | 定期 fork review agent 写内置 memory/skills；provider 在 turn/session/compress hooks 自行 ingest | frozen prompt snapshot、file lock/atomic write、provider best-effort、`<memory-context>` fence |
| Claude Code public repo | 未公开 | 未知 | 只看到文件/settings/hooks/plugins 通道 | 无法从公开 repo 判断 |

### ClawXMemory Dream 细节

ClawXMemory 的 Dream 是最接近 “AutoDream” 的实现：

1. **准备输入**：读取所有 formal project file、`_tmp` project/feedback file、现有 `project.meta.md`、user profile。
2. **全局计划**：LLM 先输出全局 project plan，判断哪些 `_tmp` 应该归到哪个正式 project，哪些项目是 alias/rename/duplicate，哪些文件该保留或删除。
3. **项目重写**：按 final project 逐个生成新的 `Project/*.md` / `Feedback/*.md` 内容。
4. **提升 `_tmp`**：把 `_tmp` 里的临时记忆 promote 到正式 `project_<hash>`。
5. **删除旧文件**：删除 absorbed / superseded / duplicate 文件。
6. **用户画像重写**：如果全局 user profile 有变化，重写 `global/User/user-profile.md`。
7. **记录 trace**：写 recent dream trace、lastDreamAt、lastDreamStatus、lastDreamSummary 等 runtime state。

它不是把所有东西总结成一个 `overview.md`，而是直接重写长期 Markdown 真源。

### Codex Phase 2 细节

Codex 的 Phase 2 是更工程化的 consolidation：

1. **DB 选输入**：从 stage-1 outputs 中按 usage/recentness 选 bounded top-N。
2. **同步 workspace**：生成 `raw_memories.md` 和 `rollout_summaries/*.md`。
3. **用 git 看变化**：如果 workspace 没变，直接成功；有变才启动 agent。
4. **写 diff 文件**：把上次 baseline 到当前 worktree 的变化写入 `phase2_workspace_diff.md`。
5. **启动受限 agent**：agent 无网络、无 approval、只可本地写 memory workspace，防止 consolidation 过程中扩散副作用。
6. **更新长期产物**：agent 根据 diff 更新 `MEMORY.md`、`memory_summary.md`、`skills/` 等。
7. **重置 baseline**：成功后把当前 workspace 设为下一次 baseline。

它的核心不是“自动归档一切”，而是“候选自动生成，长期文件改写必须可 diff、可锁、可审计”。

## 学习要点

- 如果希望人和 Agent 都能维护长期知识，ClawXMemory/Codex 的 file-memory 更容易治理。
- 如果目标是海量历史检索，MemPalace/mem0 的 vector-first 更直接，但要非常重视 scope/filter。
- SQLite 很适合做队列、状态、trace、history、lease；不适合做唯一长期语义真源。
- 真正的 AutoDream 不是“总结聊天记录”，而是“把低层候选提升、归并、删除、重写成更高层结构”。
- 一个健康的 memory pipeline 至少要回答：原始事实在哪里、候选在哪里、长期真源在哪里、谁能改、怎么回滚、怎么证明旧信息被吸收或删除。
