---
title: Memory Pipeline
type: reference
created: 2026-05-30
updated: 2026-05-30
tags:
  - life
  - stack
  - memory
  - yatoro
  - pipeline
sources:
  - "Slock task #11 #yatoro:60f8bbf7 — executable Yatoro memory pipeline design (2026-05-30)"
  - "Slock task #13 #yatoro:cdc2e737 — consolidate DB + raw text materialization v0 (2026-05-30)"
  - "Local inspection ~/.claude/projects JSONL session files (2026-05-30)"
  - "Local inspection ~/.codex/sessions JSONL rollout files (2026-05-30)"
  - "Local inspection /Users/rancune/Work/rancune/cloud/yatoro-monorepo/apps/cli (2026-05-30)"
  - "Slock thread #yatoro:60f8bbf7 msg=1ae1983f — builder structural review (2026-05-30)"
---

# Memory Pipeline

> Yatoro memory 不是每天重跑全部聊天历史，而是从多个会话来源做增量发现，只对新增或完成的片段抽取候选记忆，再经 review / gate 晋升到 wiki、skill、automation、agent MEMORY 或 code。

## Goal

Build an executable pipeline that:

1. Finds local and Slock sessions across Codex CLI, Claude Code CLI, and Slock.
2. Tracks per-source and per-session watermarks so scans are incremental.
3. Extracts candidate memories only from new / changed / completed session segments.
4. Routes candidates to the right durable artifact: wiki, skill, automation/script, agent MEMORY, code, or thread-only history.
5. Uses Git-backed wiki diffs and dry-run patches for review before long-term writes.

Non-goals for the first version:

- No vector DB / embedding retrieval.
- No background auto-Dream that rewrites wiki without review.
- No full historical daily reprocessing.
- No ingestion of unrelated product repos or private channels by default.

## Inspected Session Sources

### Claude Code CLI

Observed root:

```text
~/.claude/projects/
```

Observed file layout:

```text
~/.claude/projects/
├── -Users-rancune-Work-rancune-<project-slug>/
│   └── <session-uuid>.jsonl
├── -Users-rancune--slock-agents-<agent-id>/
│   └── <session-uuid>.jsonl
└── <project-dir>/<session-uuid>/
    └── subagents/
        └── agent-<id>.jsonl
```

Local counts on 2026-05-30:

| Kind | Count |
|---|---:|
| top-level Claude session JSONL files | 96 |
| Claude subagent JSONL files | 49 |
| total Claude JSONL files | 145 |

Observed top-level JSONL record types:

| `type` | Meaning for ingestion |
|---|---|
| `user` | Conversation input, may include text and tool result content. |
| `assistant` | Assistant output, may include text, thinking, tool use. |
| `system` | Runtime/system event; keep as metadata, do not promote directly. |
| `attachment` | Attachment metadata; keep path/ref metadata only. |
| `queue-operation` | Internal queue event; ignore for candidate extraction. |
| `mode` | Runtime mode marker; metadata only. |
| `last-prompt` | Prompt marker; metadata only. |

This list is descriptive, not exhaustive. Additional Claude record types observed by builder/code-review include `ai-title`, `file-history-snapshot`, and `permission-mode`. The adapter must be allowlist-first: only `user` / `assistant` visible text is extraction input, selected action metadata is retained, and unknown record types are ignored as metadata by default. `file-history-snapshot` can be large and should be metadata-only or dropped; never hash or extract full file snapshots.

Observed shared fields on conversation records:

```text
sessionId
uuid
parentUuid
timestamp
cwd
gitBranch
version
isSidechain
userType
message.role
message.content[]
toolUseResult
```

Observed `message.content[]` types:

| Content type | Handling |
|---|---|
| `text` | Candidate extraction input. |
| `thinking` | Do not use as durable evidence by default; keep out of wiki candidates. |
| `tool_use` | Useful as action metadata: tool name, input shape, target path. |
| `tool_result` | Potentially sensitive and verbose; truncate or exclude by default. |

Cursor strategy:

- Source id: `claude-code`.
- Session id: `sessionId` if present, else file basename without `.jsonl`.
- Locator: absolute JSONL path.
- Watermark: `{ fileSize, mtimeMs, lastByteOffset, lastUuid, lastTimestamp }`.
- Parent/child: subagent JSONL files get `parent_session_id` from path when available.
- Re-baseline if `fileSize < lastByteOffset` or an optional head hash changes; do not read from a stale offset after rewrite/truncation.
- Advance `lastByteOffset` only to the last complete newline-delimited JSON record. Preserve any half-line tail for the next scan.

Completion / segmentation:

- Claude JSONL does not expose a reliable `task_complete` event in the sampled files.
- Treat a file as processable when `mtime` is older than an idle threshold, default 15 minutes.
- Segment by appended byte range since the last offset.
- For first backfill, require explicit `--since` or `--session`; do not process all historical Claude files automatically.

Default filters:

- Include Yatoro / rancune scopes by allowlist.
- Exclude `/Users/rancune/Work/uxarts/uxarts-agent` unless explicitly requested for a uxarts task.
- Apply allowlist/denylist during discovery before parsing or queueing a file. Claude sessions from uxarts live under the same `~/.claude/projects` root, so Phase 0/1 must assert `queued uxarts sessions = 0`.
- Exclude tool outputs above size limit or matching secret-like patterns.

### Codex CLI

Observed root:

```text
~/.codex/sessions/
```

Observed file layout:

```text
~/.codex/sessions/YYYY/MM/DD/rollout-<timestamp>-<session-uuid>.jsonl
```

Local count on 2026-05-30:

| Kind | Count |
|---|---:|
| Codex rollout JSONL files | 9 |

Observed top-level JSONL record types:

| `type` | Meaning for ingestion |
|---|---|
| `session_meta` | Rollout identity, cwd, CLI version, source, model provider, git branch/commit. |
| `turn_context` | Per-turn cwd/model/sandbox/summary/timezone/turn id. |
| `response_item` | Developer/user/assistant messages, reasoning, tool calls, tool outputs, web search calls. |
| `event_msg` | Task started/completed, user/agent message mirrors, token counts, context compacted, patch/web events. |
| `compacted` | Compaction marker; metadata only. |

Observed `session_meta.payload` fields:

```text
id
cwd
timestamp
cli_version
base_instructions
source
model_provider
originator
git.branch
git.commit_hash
```

`base_instructions` may be large or sensitive. Store only presence/size/hash metadata if useful; do not feed the full value into candidate extraction.

Observed `response_item.payload.type` values:

| Payload type | Handling |
|---|---|
| `message` | Use user/assistant text as extraction input. |
| `reasoning` | Exclude from durable evidence by default. |
| `function_call` | Action metadata; keep tool name and call id. |
| `function_call_output` | Verbose/sensitive; truncate or exclude by default. |
| `custom_tool_call` | Action metadata, e.g. `apply_patch`. |
| `custom_tool_call_output` | Action result metadata; truncate by default. |
| `web_search_call` | Retrieval metadata; keep source refs if needed. |

Observed `event_msg.payload.type` values:

| Event type | Handling |
|---|---|
| `task_started` | Start a processable turn segment. |
| `task_complete` | Close a turn segment. |
| `user_message` | Human-facing input mirror; extraction input if not duplicate. |
| `agent_message` | Human-facing output mirror; extraction input if not duplicate. |
| `token_count` | Ignore for candidate extraction. |
| `context_compacted` | Metadata; useful to avoid using incomplete context as evidence. |
| `patch_apply_end` | Action metadata. |
| `web_search_end` | Source metadata. |

Cursor strategy:

- Source id: `codex-cli`.
- Session id: `session_meta.payload.id`, else UUID from filename.
- Locator: absolute rollout JSONL path.
- Watermark: `{ fileSize, mtimeMs, lastByteOffset, lastTimestamp, lastTurnId }`.
- Segment key: `turn_context.payload.turn_id` or task_started/task_complete event range.
- Re-baseline if `fileSize < lastByteOffset` or an optional head hash changes.
- Advance `lastByteOffset` only to the last complete newline-delimited JSON record.

Completion / segmentation:

- Prefer `event_msg.payload.type == task_complete` to mark a completed turn.
- If no `task_complete`, use idle threshold before extraction.
- Extract at turn granularity, not whole-file granularity, so long rollouts do not get reprocessed.

### Slock

Slock is the canonical raw memory for team conversation and task history.

Observed access surface:

```bash
slock message read --channel <target> --after <seq> --limit <n>
slock message read --channel <target> --around <idOrSeq>
slock task list --channel <channel> --status all
```

Cursor strategy:

- Source id: `slock`.
- Source locator: `#channel`, `#channel:<thread>`, `dm:@user`, or `dm:@user:<thread>`.
- Watermark: per target `{ lastSeq, lastMessageId, lastReadAt }`.
- Evidence ref: canonical Slock ref, e.g. `#yatoro:60f8bbf7`.

Completion / segmentation:

- Process task threads when task status becomes `done` or `closed`.
- Process ordinary threads when idle threshold passes and they are in an allowlisted channel.
- For current design work, default allowlist should start with `#yatoro` only.
- Private/unrelated channels are not scanned unless the user explicitly adds them to the source registry.

## Data Model

Pipeline state belongs under:

```text
~/.yatoro/db/consolidate.db
~/.yatoro/consolidate/
~/.yatoro/consolidate/raw-text/
~/.yatoro/consolidate/promotions/
```

SQLite is for watermarks, source registry, queued segments, raw-text references, and later extraction job / candidate state. It is not the long-term semantic truth. Long-term truth remains in Git-backed wiki, skills, scripts, agent MEMORY, or code.

Phase 2 uses `better-sqlite3`. Keep the state layer behind a `MemoryStateStore` interface so testing/migration code can still use fixtures, but the production store is SQLite.

Legacy Phase 1 JSON state may exist at:

```text
~/.yatoro/cache/consolidate/sources.json
~/.yatoro/cache/consolidate/sessions.json
```

If the SQLite DB is empty, import these once. Do not delete the old JSON files during migration.

### Tables

```sql
create table sources (
  id text primary key,
  kind text not null,              -- slock | codex | claude
  locator text not null,           -- root dir or slock target
  enabled integer not null default 1,
  allow_json text,
  deny_json text,
  label text,
  created_at text not null,
  updated_at text not null
);

create table watermarks (
  id text primary key,
  source_id text not null,
  external_id text not null,
  locator text not null,
  file_size integer,
  mtime_ms real,
  last_byte_offset integer,
  head_hash text,
  head_hash_len integer,
  last_seq integer,
  records_consumed integer not null default 0,
  last_scan_at text not null
);

create table segments (
  id text primary key,
  session_id text not null,
  source_id text not null,
  source_kind text not null,
  external_id text not null,
  locator text not null,
  start_offset integer,
  end_offset integer,
  start_seq integer,
  end_seq integer,
  record_count integer not null,
  hash text not null,
  status text not null,            -- queued | materialized | skipped:* | failed
  created_at text not null,
  updated_at text not null,
  unique(session_id, hash)
);

create table raw_text (
  id text primary key,
  segment_id text not null,
  path text not null,
  hash text not null,
  bytes integer not null,
  event_count integer not null,
  render_version integer not null,
  source_refs_json text,
  status text not null,            -- ready | skipped:sensitive | skipped:deferred
  created_at text not null,
  unique(segment_id, render_version)
);

create table extraction_jobs (
  id text primary key,
  raw_text_id text not null,
  segment_id text not null,
  model_id text not null,
  status text not null,            -- queued | running | done | failed | skipped:sensitive
  attempts integer not null default 0,
  last_error text,
  created_at text not null,
  updated_at text not null,
  unique(raw_text_id, model_id)
);

create table candidates (
  id text primary key,
  identity_hash text not null unique,
  scope text not null,             -- user | project | agent | capability | workflow
  kind text not null,              -- preference | decision | resource | procedure | automation_candidate | bug | design
  claim text not null,
  why_promote text,
  suggested_destination text not null,
  confidence text not null,        -- low | medium | high
  evidence_json text not null,
  status text not null,            -- inbox | accepted | rejected | promoted | superseded
  reject_reason text,
  supersedes_json text,
  created_at text not null,
  updated_at text not null
);
```

Future promotion stages will add tables such as `promotions`. Promotion state is not part of task #15 v0.

Planned future table:

```sql

create table promotions (
  id text primary key,
  candidate_id text not null references candidates(id),
  destination text not null,       -- wiki | skill | automation | agent_memory | code
  target_path text,
  patch_path text,
  status text not null,            -- draft | in_review | applied | rejected
  committed_hash text,
  created_at text not null,
  updated_at text not null
);
```

SQLite implementation notes:

- Use `PRAGMA user_version` for schema migrations.
- Wrap migration, scan segment writes, and raw text materialization writes in transactions.
- Segment upserts must tolerate re-baseline duplicates. If `session_id + hash` already exists, update metadata/status or do nothing; do not crash.
- Current implementation uses `segments.unique(session_id, hash)` with `ON CONFLICT DO NOTHING`.
- `raw_text.hash` must bind source segment content to the redaction/render version. Current implementation uses `hash(segment_hash + REDACT_RENDER_VERSION)` and stores `render_version`.
- `better-sqlite3` is the chosen backend. `yatoro consolidate doctor` must check the native binding before opening the DB; on Node ABI mismatch it should fail with a clear `pnpm rebuild better-sqlite3` hint, not crash the whole CLI.
- DB v2 adds `extraction_jobs` and `candidates`; migration moves `PRAGMA user_version` from 1 to 2 and is re-entrant.
- `inbox import` wraps candidate upserts and job completion in one transaction. It also re-checks `raw_text.status == ready` before accepting a result file.

## Normalized Event Shape

All adapters emit the same internal shape:

```ts
type MemorySourceKind = "slock" | "codex" | "claude";

interface NormalizedSession {
  sourceId: string;
  sourceKind: MemorySourceKind;
  externalId: string;
  locator: string;
  parentExternalId?: string;
  cwd?: string;
  projectKey?: string;
  startedAt?: string;
  endedAt?: string;
  fileSize?: number;
  mtimeMs?: number;
  metadata?: Record<string, unknown>;
}

interface NormalizedEvent {
  sourceId: string;
  sessionExternalId: string;
  eventId: string;                 // uuid, call_id, seq, or synthetic offset id
  parentEventId?: string;
  turnId?: string;
  timestamp?: string;
  role: "user" | "assistant" | "system" | "tool" | "event";
  kind:
    | "message"
    | "tool_call"
    | "tool_result"
    | "task_started"
    | "task_complete"
    | "attachment"
    | "metadata";
  text?: string;                   // truncated safe text only
  toolName?: string;
  locator: string;                 // file path + offset, or Slock ref
  rawRef: string;                  // evidence ref, not full raw content
  metadata?: Record<string, unknown>;
}
```

Extractor input should be built from normalized events, not raw JSONL. This gives one redaction/truncation path for all sources.

## Candidate Output Schema

LLM extraction returns strict JSON:

```json
{
  "session_ref": "codex:019e...#turn:019e...",
  "candidates": [
    {
      "scope": "project",
      "kind": "decision",
      "claim": "Yatoro memory pipeline should use per-session watermarks instead of daily full-history summarization.",
      "why_promote": "This affects implementation design and avoids repeated discussion.",
      "suggested_destination": "wiki",
      "confidence": "high",
      "evidence": [
        {
          "ref": "#yatoro:dd1350b5",
          "summary": "User asked how to avoid reprocessing all chat history."
        }
      ],
      "supersedes": []
    }
  ]
}
```

Allowed destinations:

| Destination | Use when |
|---|---|
| `discard` | No reusable value. |
| `thread_only` | Keep in raw history; searchable but not promoted. |
| `wiki` | Stable facts, resource maps, architecture decisions, workflows. |
| `skill` | Repeatable procedure with judgment. |
| `automation` | Deterministic repeatable action. |
| `agent_memory` | Startup fast path or high-frequency constraint. |
| `code` | Product/CLI should own the behavior. |

`discard` and `thread_only` are destinations, not candidate kinds. Default destination is `discard` or `thread_only`. Promote only if the candidate is reusable, reduces future cost, or affects future decisions.

## Deduplication Model

The same real task can appear in multiple sources:

- User request in Slock.
- Agent execution in a Codex or Claude CLI session.
- Final result posted back to the Slock thread.

Do not deduplicate at the source-record layer. Source records are evidence and should remain separately addressable:

```text
slock:#yatoro:342dab68
codex:<rollout-id>#turn:<turn-id>
claude:<session-id>#uuid:<uuid>
```

Deduplicate at the candidate layer:

- `identity_hash = hash(scope + kind + normalized_claim + suggested_destination)`
- `evidence_hash = hash(source_ref + source_span + evidence_summary)`

If a candidate with the same `identity_hash` already exists, append the new evidence when its `evidence_hash` is new instead of creating a second candidate. This lets Slock, Codex, and Claude reinforce the same memory without producing three durable memories.

Canonical evidence priority for promoted artifacts:

1. Slock task/thread for human-readable collaboration context.
2. Git diff/commit for code facts.
3. Codex/Claude session refs for execution details.

If two candidates are similar but not identical, keep them in inbox and require manual merge/supersede rather than fuzzy auto-merge in v1.

Cost note: candidate-layer deduplication happens after extraction. It prevents duplicate long-term memories, but it does not avoid LLM tokens spent extracting overlapping Slock / Codex / Claude segments. If overlap becomes expensive in Phase 2, add pre-extract source grouping and priority heuristics:

- If a Slock message is clearly a mirror or summary of a CLI session, extract only the canonical source.
- For task thread + CLI session pairs, prefer Slock for human intent and final decision, and use CLI only for execution details.
- Group nearby sources by time window, `cwd`, and task/thread ref before extraction.

## CLI Surface

Chosen Phase 1 command namespace:

```bash
yatoro consolidate <subcommand>
```

Rationale: the CLI already has `yatoro memo` for flomo notes. `yatoro memory` would sit next to `memo` in shell completion and create user confusion. `consolidate` names the workflow: turn raw sessions into reviewed long-term assets.

Recommended command surface:

```bash
yatoro consolidate sources list
yatoro consolidate sources add <kind> <locator> [--label <label>] [--allow <path-or-target>]
yatoro consolidate sources disable <id>

yatoro consolidate scan [--source <id>] [--since <date>] [--idle-minutes <n>] [--dry-run] [--json]
yatoro consolidate materialize [--limit <n>] [--source <id>] [--dry-run] [--json]
yatoro consolidate extract packet [--limit <n>] [--source <id>] [--dry-run] [--json]
yatoro consolidate inbox list [--scope <scope>] [--destination <dest>] [--status inbox] [--json]
yatoro consolidate inbox show <candidate-id>
yatoro consolidate inbox reject <candidate-id> [--reason <text>]
yatoro consolidate inbox import <result-file> [--json]

yatoro consolidate promote <candidate-id> --destination <dest> [--target <path>] [--dry-run] [--json]
yatoro consolidate promotion apply <promotion-id>

yatoro consolidate doctor [--json]
```

Command defaults:

- `scan` is cheap and never calls an LLM.
- `materialize` converts queued source segments into cleaned model-ready raw text files; it never calls an LLM.
- `extract packet` emits handoff packets for the assigned extraction worker. After `@yatoro-memory-extractor` was removed from #yatoro, the temporary worker is `@yatoro-code-review` by explicit coordinator/user assignment. The CLI does not call a model, SDK, or external API.
- `inbox import` is the only candidate inbox writer. It validates worker result JSON, applies anti-copy/sensitivity gates, computes identity hashes, and merges duplicate candidates.
- `promote` defaults to `--dry-run`; it creates a patch or draft file but does not apply it unless explicitly requested.
- `doctor` checks the `better-sqlite3` native binding, DB/work/raw-text paths, legacy JSON migration state, source discovery, and DB counts. If the native binding fails to load, it must print a rebuild hint and exit non-zero instead of crashing.

## Adapter Implementation Plan

Target code location:

```text
/Users/rancune/Work/rancune/cloud/yatoro-monorepo/apps/cli/src/commands/consolidate/index.ts
/Users/rancune/Work/rancune/cloud/yatoro-monorepo/apps/cli/src/lib/consolidate/
```

These are TypeScript source paths. Runtime ESM imports should follow the existing CLI pattern and import compiled `.js` module specifiers.

Task #13 concrete CLI layout:

```text
src/config.ts
src/commands/consolidate/index.ts
src/lib/consolidate/
  types.ts
  jsonl.ts
  scope.ts
  state/{store.ts,schema.ts}
  adapters/{claude.ts,codex.ts,slock.ts}
  text/{normalize.ts,redact.ts,render.ts}
  extract/{candidate.ts,packet.ts,import.ts}
  scan.ts
  materialize.ts
  doctor.ts
```

Add config constants:

```ts
export const CONSOLIDATE_DIR = join(YATORO_DIR, "consolidate");
export const CONSOLIDATE_RAWTEXT_DIR = join(CONSOLIDATE_DIR, "raw-text");
export const CONSOLIDATE_JOBS_DIR = join(CONSOLIDATE_DIR, "extraction-jobs");
export const CONSOLIDATE_RESULTS_DIR = join(CONSOLIDATE_DIR, "extraction-results");
export const CONSOLIDATE_DB_DIR = join(YATORO_DIR, "db");
export const CONSOLIDATE_DB_FILE = join(CONSOLIDATE_DB_DIR, "consolidate.db");
export const LEGACY_CONSOLIDATE_SOURCES_FILE = join(CACHE_DIR, "consolidate", "sources.json");
export const LEGACY_CONSOLIDATE_STATE_FILE = join(CACHE_DIR, "consolidate", "sessions.json");
```

Register command from `src/index.ts` the same way `skills` is registered.

## Source Registry Defaults

Initial defaults should be explicit and conservative:

```json
{
  "version": 1,
  "sources": [
    {
      "id": "claude-code",
      "kind": "claude",
      "locator": "/Users/rancune/.claude/projects",
      "enabled": true,
      "allow": [
        "/Users/rancune/Work/rancune",
        "/Users/rancune/.slock/agents"
      ],
      "deny": [
        "/Users/rancune/Work/uxarts/uxarts-agent"
      ]
    },
    {
      "id": "codex-cli",
      "kind": "codex",
      "locator": "/Users/rancune/.codex/sessions",
      "enabled": true,
      "allow": [
        "/Users/rancune/Work/rancune",
        "/Users/rancune/.slock/agents"
      ]
    },
    {
      "id": "slock-yatoro",
      "kind": "slock",
      "locator": "#yatoro",
      "enabled": true
    }
  ]
}
```

Do not scan all visible channels by default. Add channels/DMs only by explicit user instruction.

## Redaction Rules

Before materializing raw text or extraction:

1. Drop or truncate tool outputs by default.
2. Keep tool names, target paths, exit status, and short summaries.
3. Drop `reasoning`, Claude `thinking`, encrypted content, credentials, auth files, and secret-looking values.
4. Cap event text length, for example 8 KB per event and 64 KB per segment.
5. Never write raw transcript text into wiki as evidence; write source refs and short evidence summaries.

Raw text materialization has a stricter boundary: redaction must run before any write to `~/.yatoro/consolidate/raw-text/`, and the redact -> render path must be the only writer. There must be no code path that writes unredacted raw event text to disk.

Sensitive paths:

```text
~/.yatoro/auth.json
*.pem
*.key
.env
id_rsa
id_ed25519
```

If a segment contains sensitive-looking content, materialization should skip it and mark `raw_text.status = skipped:sensitive`; do not write a redacted version by default. Candidate extraction should then skip or require manual handling.

Filesystem permissions:

- `~/.yatoro/consolidate/` and `raw-text/` directories must be created with mode `0700`.
- Raw text files must be created with mode `0600`.
- Treat raw text with the same sensitivity as local auth/session material.

## Scan Algorithm

Pseudo-flow:

```text
load sources
for each enabled source:
  adapter.discover()
  for each discovered session:
    if not allowed by scope filters:
      mark ignored
      continue
    compare file size / mtime / seq with stored cursor
    if unchanged:
      skip
    if active and not idle:
      update session metadata only
      continue
    if fileSize < lastByteOffset or head hash changed:
      re-baseline this file and do not read from the old offset
      continue unless explicit --backfill-rewritten is set
    read only appended byte range or --after seq
    keep only complete newline-delimited JSON records
    buffer incomplete tail; do not advance watermark past it
    create source segment from the appended byte/seq range
    hash the source segment
    upsert segment if hash is new
    leave segment queued for materialize
```

Watermark advances after the queued segment is safely persisted. Materialize/extract failure must not lose cursor state; the segment remains queued/failed and can be retried idempotently.

## Materialize Algorithm

`materialize` is the first extraction layer. It converts source-specific records into cleaned, model-ready text without calling an LLM.

Pseudo-flow:

```text
select queued segments
for each segment:
  if source is slock:
    mark skipped:deferred until Slock M2 pagination/range gate is cleared
    continue
  read source byte range from Claude/Codex JSONL
  parse complete JSONL records
  normalize to NormalizedEvent:
    - user/assistant visible text
    - tool_call metadata only
    - no reasoning/thinking
    - no full tool_result output
  redact before any disk write
  render compact text:
    [user] ...
    [assistant] ...
    [tool: Edit -> <target>] (output omitted)
  if rendered text or source refs look sensitive:
    mark skipped:sensitive
    do not write a raw text file
    continue
  write ~/.yatoro/consolidate/raw-text/<source>/<segment-id>.txt with mode 0600
  insert raw_text row with path/hash/bytes/event_count/render_version/source_refs/status
```

`materialize --dry-run` must return the planned actions without creating raw text files or DB rows.

## Extract Algorithm

Use a Yatoro-specific extractor. Do not directly reuse Codex or Hermes extraction code as the implementation.

Design references:

- Codex: use the architectural pattern of stage-1 extraction followed by reviewed consolidation. Do not copy its destination model, because Yatoro routes to wiki, skill, automation, agent MEMORY, and code rather than a single Codex memory workspace.
- Hermes: use the idea of lifecycle hooks / providers as inspiration for when extraction can run. Do not use its provider write model as the source of truth; Yatoro's durable truth remains Git-backed wiki, skills, scripts, MEMORY, and code.
- Hermes providers such as mem0 / Honcho / Supermemory are vector-first or external-service-first. They can be considered later as optional retrieval layers, but they should not become Yatoro's canonical long-term memory backend.
- Codex's consolidated artifacts, such as `~/.codex/memories/MEMORY.md`, can be considered later as high-level sources. Phase 2 v1 should extract from raw sessions first so candidates share one schema.

Extractor principle:

- Extract candidates, not long summaries.
- Prefer no candidate over noisy candidates.
- Default to `discard` or `thread_only`.
- Never write durable artifacts during extract.
- In task #15 v0, extraction is agent-worker based: the CLI emits packets, an explicitly assigned worker reads them and writes result JSON, and the CLI imports those results. The CLI does not call an LLM provider. Current temporary worker: `@yatoro-code-review` (after `@yatoro-memory-extractor` was removed from #yatoro).
- In task #16, extraction quality is driven by explicit guidance: keep durable user preferences, decisions with reasons, stable domain/project facts, and reusable working methods; discard pure tool/runtime meta, temporary debug/progress, and chatter.

Pseudo-flow:

```text
extract packet:
  select raw_text.status=ready rows not already done/skipped for the extractor model id
  read raw text and run egress looksSensitive gate
  if sensitive:
    record extraction_job status skipped:sensitive
    do not emit packet
  else:
    write 0600 job packet to ~/.yatoro/consolidate/extraction-jobs/<raw_text_id>.json
    packet includes raw_text_path, source refs, constraints, and result_path

worker:
  read packet + raw_text_path
  perform semantic extraction, not quoting/copying raw text
  write 0600 result JSON to result_path under ~/.yatoro/consolidate/extraction-results/

inbox import:
  read result JSON
  verify raw_text exists and status is still ready
  validate schema, destination/confidence allowlists, length limits, anti-copy patterns, and sensitivity
  apply meta keyword gate:
    if tooling/meta terms appear in an otherwise valid candidate, downgrade destination to thread_only and confidence to low
    do not hard-reject solely for tooling/meta terms
  compute identity_hash from scope + kind + normalized claim + destination
  insert candidate, or merge new evidence into existing identity_hash
  mark extraction_job done in the same transaction as candidate writes
```

Candidate quality rules:

- Prefer no candidate over noisy candidate.
- Reject claims that only summarize task chatter.
- Require evidence refs for every promoted candidate.
- Require a destination rationale for `wiki`, `skill`, `automation`, `agent_memory`, or `code`.
- Candidate text should be semantic extraction, not raw quotes. `claim` and `why_promote` may be up to about 800 characters each; `evidence.summary` may be up to about 500 characters. Import rejects code blocks, rendered chat markers, too many pasted lines, secret-like strings, invalid destinations, and invalid confidence values.
- Quality gate split: semantic guidance should discard pure meta; tooling/meta keywords only downgrade to `thread_only`, preserving edge cases where a real user preference or architecture decision mentions infrastructure terms such as `slock.ai`, `render_version`, or `watermark`. Sensitive content, anti-copy violations, and invalid schema remain hard rejects.

### Agent-Worker Packet Contract

Job packet generated by the CLI:

```json
{
  "version": 1,
  "job_id": "extract:<raw_text_id>",
    "worker": "yatoro-code-review",
  "segment_id": "...",
  "raw_text_id": "...",
  "raw_text_path": "/Users/rancune/.yatoro/consolidate/raw-text/...txt",
  "result_path": "/Users/rancune/.yatoro/consolidate/extraction-results/<raw_text_id>.json",
  "source": {
    "source_id": "codex-cli",
    "source_kind": "codex",
    "source_refs": ["codex:<session>#offset:<start>-<end>"]
  },
  "constraints": {
    "no_promotion": true,
    "no_raw_text_in_output": true,
    "semantic_extraction_only": true,
    "extraction_guidance": "Extract only durable user knowledge, preferences, decisions with reasons, stable facts, and reusable working methods. Discard pure tool/runtime meta, temporary debug/progress, and chatter. When uncertain, prefer no candidate or thread_only.",
    "allowed_destinations": ["discard", "thread_only", "wiki", "skill", "automation", "agent_memory", "code"],
    "default_destination": "thread_only"
  }
}
```

Result JSON written by the assigned worker:

```json
{
  "version": 1,
  "job_id": "extract:<raw_text_id>",
  "worker": "yatoro-code-review",
  "candidates": [
    {
      "scope": "project",
      "kind": "knowledge",
      "claim": "...",
      "why_promote": "...",
      "suggested_destination": "wiki",
      "confidence": "high",
      "evidence": [{ "ref": "...", "summary": "..." }],
      "supersedes": []
    }
  ]
}
```

Result files must not contain raw text, long quotes, full tool output, code/log blocks, or secret-like text. Evidence is `{ref, summary}` only.

## Promote Algorithm

Promotion must be dry-run first:

```text
load candidate
resolve destination target
generate patch/draft
write patch to ~/.yatoro/consolidate/promotions/<id>.patch
show diff and target path
mark promotion draft
```

Apply only after explicit command:

```text
yatoro consolidate promotion apply <promotion-id>
```

Destination behavior:

| Destination | Action |
|---|---|
| `wiki` | Patch a Git-backed wiki page; update frontmatter `updated:` and `sources:`. |
| `skill` | Patch `SKILL.md` under agent-kit; keep procedural style. |
| `automation` | Create or patch script/CLI proposal; implementation still needs Builder. |
| `agent_memory` | Patch only a small fast-path pointer; never paste long notes. |
| `code` | Create a task/proposal; do not auto-edit product code from memory promotion. |

Use `superseded_by` / `supersedes` markers instead of hard deletion when replacing old wiki sections.

## Implementation Phases

### Phase 0 — fixtures and adapter proof

Deliverables:

- Sanitized fixture JSONL lines for Claude and Codex under CLI tests.
- Adapter tests proving session discovery and field parsing.
- `consolidate doctor` reports discovered source counts and unreadable files.

Commands:

```bash
pnpm --filter @yatoro/cli typecheck
pnpm --filter @yatoro/cli build
yatoro consolidate doctor --json
```

Acceptance:

- Claude adapter sees top-level and subagent JSONL files.
- Codex adapter reads date-partitioned rollout files.
- No raw message content is printed by doctor.
- Discovery-stage denylist works before parsing/queueing; `queued uxarts sessions = 0` under default source registry.

### Phase 1 — scan only, no LLM

Status: implemented in task #12.

Deliverables:

- `yatoro consolidate sources list`
- `yatoro consolidate scan --dry-run`
- State store with watermarks and queued source segments.

Acceptance:

```bash
yatoro consolidate scan --source claude-code --dry-run --json
yatoro consolidate scan --source codex-cli --dry-run --json
yatoro consolidate scan --source slock-yatoro --dry-run --json
```

- Re-running scan with no changes returns zero queued segments.
- Appending to a fixture queues only the appended segment.
- Default scan must not queue any session whose `cwd` or project slug resolves to `/Users/rancune/Work/uxarts/uxarts-agent`.
- Truncated or rewritten fixture is re-baselined and does not read from a stale byte offset.
- Fixture ending with a half-written JSONL line leaves that tail unprocessed and does not advance the watermark past it.
- Slock `seq=` parsing is anchored to canonical message header lines, not message body text.

### Phase 2 — materialize first, then extract candidates

Phase 2 is split because raw-text materialization is deterministic and reviewable, while candidate extraction introduces an LLM.

Prerequisites for candidate extraction:

- Clear M2 for Slock: prove `slock message read --after <seq> --limit <n>` paginates forward continuously from the cursor, not as a latest-window query. Add a fixture or integration test with more than 200 new messages so Slock extraction cannot silently skip the range between the old cursor and the returned window start.
- Choose the LLM runner mode. Recommended first mode is agent-driven extraction; CLI-embedded extraction comes later.
- Tighten shell/exec tool target redaction before feeding raw text to an LLM. Current v0 keeps only the first command line up to a short cap; Phase 2 should either keep only the tool name or run `looksSensitive` on target text before extraction.

### Phase 2a — DB and raw text materialization

Status: implemented in task #13.

Deliverables:

- Move consolidate working files to `~/.yatoro/consolidate/`.
- Use `better-sqlite3` with DB at `~/.yatoro/db/consolidate.db`.
- Import legacy `~/.yatoro/cache/consolidate/{sources,sessions}.json` once if DB is empty.
- Add `yatoro consolidate materialize [--limit <n>] [--dry-run]`.
- Materialize cleaned model-ready raw text to `~/.yatoro/consolidate/raw-text/<source>/<segment-id>.txt`.
- DB stores source refs, segment metadata, raw text path, hash, render version, bytes, event count, and status.

Acceptance:

```bash
yatoro consolidate doctor --json
yatoro consolidate scan --source codex-cli --json
yatoro consolidate materialize --limit 1 --dry-run --json
yatoro consolidate materialize --limit 1 --json
```

- `materialize --dry-run` writes no raw text file and no `raw_text` DB row.
- Raw text files are mode `0600`; parent dirs are mode `0700`.
- Raw text contains only redacted/rendered model-ready text; no reasoning/thinking, full tool output, or secret-like content.
- Sensitive file-path fixtures such as `.env`, `.pem`, `.key`, `id_rsa`, and `id_ed25519` are marked `skipped:sensitive`, including when the sensitive path appears inside a rendered tool-call line.
- Sensitive fixture is marked `skipped:sensitive` and does not write a raw text file.
- `raw_text.render_version` is stored and included in hash/re-generation logic.
- Legacy JSON import is idempotent and does not delete old JSON files.
- Phase 1 smoke tests remain passing as regression coverage.
- `yatoro consolidate doctor` reports `binding: better-sqlite3 ok`; if the native binding cannot load, it exits non-zero with a `pnpm rebuild better-sqlite3` hint.

### Phase 2b — extract candidates

Status: implemented in task #15.

Deliverables:

- `yatoro consolidate extract packet --limit <n>`
- Candidate schema validation.
- `yatoro consolidate inbox import/list/show/reject`.
- Reusable manual/daily driver script: `apps/cli/scripts/consolidate-extract-one.sh`.

Acceptance:

```bash
yatoro consolidate extract packet --limit 1 --dry-run --json
yatoro consolidate extract packet --limit 1 --json
yatoro consolidate inbox import ~/.yatoro/consolidate/extraction-results/<raw_text_id>.json --json
yatoro consolidate inbox list --json
```

- Extraction never writes wiki/skill/MEMORY.
- CLI never calls a model provider; worker execution is outside CLI and mediated by packet/result files.
- `extract packet --dry-run` writes no files or DB rows.
- Packet files are mode `0600`; extraction job/result directories are mode `0700`.
- `extract packet` re-runs the sensitivity gate before handing raw text to the worker.
- Candidate evidence refs point back to source refs.
- Duplicate `identity_hash` is merged with appended evidence, not duplicated.
- `inbox import` is the only candidate writer; it rejects overlong fields, anti-copy patterns, secret-like strings, bad destinations, bad confidence values, and results whose `raw_text.status` is not `ready`.
- `inbox import` downgrades tooling/meta keyword hits to `thread_only` rather than hard-rejecting them. This keeps the inbox clean for promotion while avoiding silent loss of real user preferences or architecture decisions that mention infrastructure terms.
- `inbox reject` is a status update with `reject_reason`; it does not delete candidates.
- Daily/manual script `consolidate-extract-one.sh prepare [limit]` clears only `extraction-jobs/` and `extraction-results/`, never DB/watermarks/raw_text.
- Quality smoke baseline after task #16: meta idle-filter sample yields no durable candidate; keep fixture preserves user preferences/decisions such as flexible review, filtering tool meta, agent-worker/no-SDK v0, and semantic synthesis without copying source text.

### Phase 3 — promotion dry-run

Deliverables:

- `yatoro consolidate promote <candidate-id> --destination wiki --dry-run`
- Draft patch files under `~/.yatoro/consolidate/promotions/`.
- Wiki target resolver for stack pages.

Acceptance:

```bash
yatoro consolidate promote <candidate-id> --destination wiki --target wiki/life/stack/memory-pipeline.md --dry-run
git -C /Users/rancune/Work/rancune/stardust diff --check
```

- Dry-run creates a patch but does not modify files.
- Apply updates frontmatter `updated:` and `sources:`.

### Phase 4 — hooks and scheduled sweep

Deliverables:

- Optional Claude/Codex hook docs or scripts that run `yatoro consolidate scan --source ...`.
- Slock-triggered scan after task close.
- Low-frequency sweep as a fallback.

Acceptance:

- Session-end/hook scan processes one source/session only.
- Scheduled sweep only scans metadata and queued deltas.
- No daily full-history extraction.

### Phase 5 — retrieval integration

Deliverables:

- Coordinator lookup recipe:
  1. wiki stack map
  2. skills index
  3. memory inbox / promoted candidates
  4. Slock search
  5. Codex/Claude session index
- Optional `yatoro consolidate search` over candidates and source metadata.

Acceptance:

- A future request can retrieve candidate refs without loading raw transcripts.
- Search result shows source refs and destination status.

## Review Gates

Use @yatoro-code-review before applying:

- New source adapter that reads private/session logs.
- Extractor prompt/schema changes.
- Any command that writes wiki/skills/MEMORY/code.
- Any automatic hook or scheduled job.
- Any change to allowlist/denylist defaults.

## Closed Decisions

1. Command namespace is `yatoro consolidate`, not `yatoro memory`, to avoid confusion with existing `yatoro memo`.
2. Production state backend is `better-sqlite3` at `~/.yatoro/db/consolidate.db`.
3. Working files live under `~/.yatoro/consolidate/`, not `~/.yatoro/cache/`.
4. Raw text is file-backed under `~/.yatoro/consolidate/raw-text/`; DB stores metadata, hashes, status, and paths.
5. Legacy JSON under `~/.yatoro/cache/consolidate/` is imported once when the DB is empty and is not deleted by migration.
6. Candidate extraction v0 uses an agent-worker via packet/result files. The CLI does not embed Anthropic SDK/fetch or manage model API keys.
7. `@yatoro-memory-extractor` was removed from #yatoro. Until a dedicated extractor is restored, `@yatoro-code-review` can be explicitly assigned as the temporary extraction worker. The worker only processes explicit extraction job packets and never promotes or writes durable assets.

## Open Decisions

1. CLI-embedded extraction:
   - Current implementation is agent-worker first.
   - A future CLI-embedded model backend would require explicit review for API keys, egress policy, provider errors, and unattended operation.
2. Promotion approval:
   - Coordinator apply after review, or human-only apply for wiki/skill writes.
3. Backfill policy:
   - Default should be no historical backfill.
   - Use explicit `--since` / `--session` / `--source` for migration.

## Completed Execution Tasks

Task #12:

> Implement `yatoro consolidate` scan v0: sources/doctor/scan --dry-run, Claude/Codex/Slock adapters, watermarks; no LLM, no durable content writes.

Task #13:

> Implement consolidate DB + raw text materialization v0: `~/.yatoro/consolidate/`, `~/.yatoro/db/consolidate.db`, `better-sqlite3`, legacy JSON migration, segment table, redacted raw text files, and `materialize`.

Task #13 verification baseline:

- `pnpm --filter @yatoro/cli typecheck`
- `pnpm --filter @yatoro/cli build`
- `bash apps/cli/scripts/consolidate-smoke.sh` — 23/23 pass at review sign-off.

Task #14:

> Harden consolidate permissions: `~/.yatoro/consolidate` is chmod 0700; `~/.yatoro/db/consolidate.db` and sidecars are chmod 0600; shared `~/.yatoro/db` and `yatoro.db` are not changed.

Task #15:

> Implement Phase 2b extract candidate v0 in agent-worker mode: DB v2, packet/result handoff, `inbox import/list/show/reject`, identity-hash dedup, anti-copy/sensitivity gates, and manual/daily helper script.

Task #15 verification baseline:

- `pnpm --filter @yatoro/cli typecheck`
- `pnpm --filter @yatoro/cli build`
- `bash apps/cli/scripts/consolidate-smoke.sh` — 38/38 pass at review sign-off.
- Real end-to-end test: packet `38930bb3-9e1a-4771-90f9-034fb07c437e` -> worker result -> `inbox import` -> 2 inbox candidates.

Task #16:

> Improve extraction quality with explicit guidance, meta keyword downgrade, and evaluator comparison. The comparison showed a lightweight worker is sufficient when guidance is strong; after `@yatoro-memory-extractor` was removed, `@yatoro-code-review` can temporarily perform extraction by explicit assignment.

Task #16 verification baseline:

- `pnpm --filter @yatoro/cli typecheck`
- `pnpm --filter @yatoro/cli build`
- `bash apps/cli/scripts/consolidate-smoke.sh` — 46/46 pass at builder report.
- Comparison experiment:
  - Sample A, consolidate idle-filter meta: lightweight extractor and `@yatoro-code-review` both returned 0 durable candidates.
  - Sample B, real user preferences/decisions fixture: both returned 4 high-value semantic candidates with matching evidence refs and no raw-text copying.
- Final gate policy:
  - semantic guidance discards pure meta,
  - tooling/meta keywords downgrade to `thread_only`,
  - sensitive, anti-copy, and invalid schema remain hard reject.
