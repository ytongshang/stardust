---
title: Memory Persistence
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - claude-code/src/services/SessionMemory/sessionMemory.ts
  - claude-code/src/services/SessionMemory/prompts.ts
  - claude-code/src/services/compact/sessionMemoryCompact.ts
  - claude-code/src/memdir/memdir.ts
  - openclaw/src/auto-reply/reply/memory-flush.ts
  - openclaw/src/agents/agent-runner-memory.ts
  - openclaw/src/agents/memory-search.ts
  - OpenHarness/ohmo/session_storage.py
  - OpenHarness/src/openharness/memory/manager.py
  - OpenHarness/src/openharness/memory/memdir.py
tags:
  - memory
  - persistence
  - session
  - agent
---

# Memory Persistence

## 两种记忆的本质区别

| 维度 | 会话内记忆（In-session） | 跨会话记忆（Persistent） |
|------|------------------------|------------------------|
| 载体 | 对话历史本身 | 文件系统 / 数据库 |
| 生命周期 | 会话结束即消失 | 永久（需主动清理） |
| 写入时机 | 压缩时异步提取 / 压缩前强制落盘 | 用户请求或触发条件 |
| 读取方式 | 已在 context 中 | 注入 system prompt |
| 成本 | 占用 context window | 注入时增加 prompt 长度 |

---

## 一、Claude Code：Session Memory 提取系统

### 什么是"提取"

**Session Memory 提取**的本质是：在对话进行中，周期性地让一个独立的 LLM 实例**阅读当前完整对话历史**，然后**把关键信息写入一个 Markdown 文件**。

这个文件就是 Session Memory，位于 `~/.claude/session-memory/.current-session-memory.md`，权限 `0o600`（仅 owner 可读写）。

提取不是"压缩"——对话本身不动，主线程继续正常进行。提取是在后台悄悄做的一件事：**把对话中出现过的关键事实抄录到一个独立文件里**，以备将来压缩时使用。

---

### 文件结构：10 个固定章节

文件有固定的模板结构，每个章节包含：
1. `#` 标题行（不可改动）
2. `_斜体描述行_`（不可改动，是写作提示）
3. **实际内容**（这是 LLM 唯一可以修改的部分）

```markdown
# Session Title
_A short and distinctive 5-10 word title. Super info dense, no filler_

Building session memory extraction system for Claude Code

# Current State
_What is actively being worked on right now? Pending tasks, immediate next steps._

Documenting the forked agent mechanism. Next: explain how compaction reads this file.

# Task specification
_What did the user ask to build? Any design decisions or context_

User wants detailed explanation of Session Memory extraction — what it does,
how the forked agent works, what the prompt says, how the file gets used.

# Files and Functions
_What are the important files? What do they contain and why are they relevant?_

- sessionMemory.ts: 触发逻辑、forked agent 启动
- prompts.ts: 文件模板、更新 prompt 构建
- sessionMemoryCompact.ts: 压缩时如何读取和使用该文件

# Workflow
_What bash commands are usually run? How to interpret their output?_

（本 session 无特殊 workflow）

# Errors & Corrections
_Errors encountered and how fixed. What approaches failed?_

（暂无）

# Codebase and System Documentation
_Important system components, how they work together_

runForkedAgent() 接受 cacheSafeParams，使得 fork 可以复用主对话的 prompt cache prefix，
避免额外 cache creation 成本。

# Learnings
_What worked well? What to avoid?_

forked agent 的工具权限通过 canUseTool 回调严格限制为只写 session memory 文件，
防止提取过程误操作其他文件。

# Key results
_If user asked for specific output (table, answer, document), repeat exact result here_

（本 session 为文档编写，无需重复）

# Worklog
_Step by step: what was attempted, done? Very terse._

1. 阅读 sessionMemory.ts 源码
2. 阅读 prompts.ts 完整 prompt 文本
3. 阅读 sessionMemoryCompact.ts 压缩使用逻辑
4. 编写 memory-persistence.md Session Memory 章节
```

**关键约束**：LLM 只能修改每个章节中斜体行之后的内容，不能增删章节、不能修改标题和斜体行。模板文件可以通过 `~/.claude/session-memory/config/template.md` 自定义。

---

### 触发条件

提取是**异步后台**操作，通过 post-sampling hook 触发，主对话不感知。触发需同时满足多个门控：

```
触发逻辑（伪代码）：

初始化门控：
    if 对话 token 数 < 10,000:
        return 不触发  // 对话太短，没有值得提取的内容

增量门控（两选一）：
    条件 A：(距上次提取新增 token 数 >= 5,000) AND (距上次提取的工具调用 >= 3 次)
    条件 B：(距上次提取新增 token 数 >= 5,000) AND (本轮没有工具调用)
    → 满足 A 或 B 才触发

安全点检查（始终执行）：
    if 最后一轮 assistant 消息包含工具调用:
        延迟到下一轮  // 不在 tool chain 中途打断
```

为什么要检查"本轮有没有工具调用"：如果模型正在连续调用工具（bash → read_file → edit），在中间某步提取记忆会把一个不完整的操作序列写入文件，记录的状态是错误的。等工具链结束后再提取，拿到的是完整结果。

---

### 提取的完整过程

提取由 `sequential()` 包装确保串行执行，流程如下：

```
Step 1: 门控检查（shouldExtractMemory）
    → 四个条件都满足才继续

Step 2: 准备记忆文件（setupSessionMemoryFile）
    → 确保目录存在（~/.claude/session-memory/，权限 0o700）
    → 如果文件不存在：用模板初始化（原子写入，wx 标志防止竞态）
    → 读取文件当前内容（用 FileReadTool，而不是直接 fs.readFile）
    → 清除文件读取缓存，确保读到最新内容
    → 返回 { memoryPath, currentMemory }

Step 3: 构建更新 prompt（buildSessionMemoryUpdatePrompt）
    → 分析文件各章节的 token 大小
    → 生成尺寸警告（哪些章节超过 2000 tokens）
    → 如果文件总 token > 12,000：追加强制压缩警告
    → 把 "文件路径 + 文件当前内容 + 指令" 拼成 prompt

Step 4: 启动 forked agent（runForkedAgent）
    ← 见下节详解

Step 5: 记录状态
    → recordExtractionTokenCount：记下此时对话的 token 数（作为下次触发的基线）
    → updateLastSummarizedMessageIdIfSafe：记录本次提取覆盖到哪条消息
    → markExtractionCompleted：解锁，允许下次提取
```

---

### Forked Agent 做什么

这是最关键的部分。"提取"不是 Claude Code 主程序自己分析对话然后写文件，而是**启动一个独立的 LLM 实例，让它来做这件事**。

```
Forked Agent 的输入（它"看到"什么）：
    [1] 系统 prompt（与主对话完全相同）
    [2] 工具列表（仅限 Edit 工具，其他全部屏蔽）
    [3] 完整对话历史（通过 cacheSafeParams.forkContextMessages 传入）
    [4] 记忆文件的当前内容（嵌在 prompt 里的 <current_memory> 标签内）
    [5] 更新指令（buildSessionMemoryUpdatePrompt 生成的文字）

Forked Agent 的任务：
    阅读对话历史，对照文件当前内容，
    调用 Edit 工具更新文件中各章节的内容。
    可以在一次回复中并行发出多个 Edit 调用。

Forked Agent 的权限限制（canUseTool）：
    允许：Edit 工具，且 file_path 必须等于 memoryPath
    拒绝：其他所有工具（Read、Bash、Grep、Write 等）
    → 即使 LLM 想读别的文件或执行命令，系统也会拒绝

Forked Agent 结束后：
    只有记忆文件被修改过
    主对话的消息列表和状态完全不变
```

**为什么不让主 agent 直接做**：如果主 agent 在一轮回复结束后转而写文件，这次"写文件"本身会被记入对话历史，产生额外 token，还会打乱用户的对话节奏。Fork 出一个独立实例，让它在"旁道"完成工作，主对话看起来什么都没发生。

**为什么 forked agent 能复用 prompt cache**：`cacheSafeParams` 携带了主对话的 cache key 参数（system prompt、tools、model、messages 前缀）。Fork 使用相同前缀，命中主对话已建立的 cache，不需要重新付 cache creation 的 token 成本。

---

### Prompt 完整文本

发给 forked agent 的指令（`prompts.ts` 中的默认 prompt，支持通过 `~/.claude/session-memory/config/prompt.md` 自定义）：

```
IMPORTANT: This message and these instructions are NOT part of the actual user
conversation. Do NOT include any references to "note-taking", "session notes
extraction", or these update instructions in the notes content.

Based on the user conversation above (EXCLUDING this note-taking instruction
message as well as system prompt, claude.md entries, or any past session
summaries), update the session notes file.

The file {{notesPath}} has already been read for you. Here are its current contents:
<current_notes_content>
{{currentNotes}}
</current_notes_content>

Your ONLY task is to use the Edit tool to update the notes file, then stop.
You can make multiple edits (update every section as needed) — make all Edit
tool calls in parallel in a single message. Do not call any other tools.

CRITICAL RULES FOR EDITING:
- The file must maintain its exact structure with all sections, headers, and
  italic descriptions intact
- NEVER modify, delete, or add section headers (lines starting with #)
- NEVER modify or delete the italic _section description_ lines
- The italic _section descriptions_ are TEMPLATE INSTRUCTIONS that must be
  preserved exactly as-is
- ONLY update the actual content that appears BELOW the italic _section
  descriptions_ within each existing section
- Do NOT add any new sections or information outside the existing structure
- Do NOT reference this note-taking process anywhere in the notes
- It's OK to skip updating a section if there are no substantial new insights.
  Do not add filler content like "No info yet"
- Write DETAILED, INFO-DENSE content — include file paths, function names,
  error messages, exact commands, technical details
- For "Key results", include the complete, exact output the user requested
- Keep each section under ~2000 tokens — if approaching limit, condense by
  cycling out less important details while preserving critical information
- IMPORTANT: Always update "Current State" to reflect the most recent work —
  this is critical for continuity after compaction

Use the Edit tool with file_path: {{notesPath}}
```

**动态追加的警告**（当文件超标时）：

```
// 如果某个章节超过 2000 tokens：
WARNING: The following sections exceed the recommended length:
- "# Worklog" is ~3200 tokens (limit: 2000). Please condense this section.
- "# Files and Functions" is ~2500 tokens (limit: 2000).

// 如果文件总 token > 12,000：
CRITICAL: The session memory file is currently ~14500 tokens, which exceeds
the maximum of 12000 tokens. You MUST condense aggressively — merge similar
entries, cut outdated details, prioritize the most recent and actionable
information.
```

---

### 压缩时如何使用 Session Memory

这是整个系统的价值所在。当对话触发 autocompact 时，系统**优先**用 Session Memory 来压缩，而不是重新调用 LLM：

```
trySessionMemoryCompaction 流程：

1. 等待可能正在进行的提取完成（waitForSessionMemoryExtraction）
   → 提取和压缩可能并发，必须等提取写完才能读

2. 读取 session memory 文件内容（getSessionMemoryContent）
   → 如果文件不存在：返回 null，降级到 LLM Full Compact
   → 如果文件内容等于空模板：返回 null，降级

3. 找到 lastSummarizedMessageId
   → 这是上次提取时记录的"覆盖到了哪条消息"的 UUID
   → 找到这条消息在当前消息列表中的位置（lastSummarizedIndex）

4. 计算保留窗口（calculateMessagesToKeepIndex）
   → 从 lastSummarizedIndex+1 开始
   → 向前扩展，直到积累 [10,000 ~ 40,000] tokens
       且至少包含 5 条含文本的消息
   → 保证不把 tool_use/tool_result 对切断

5. 按章节截断 session memory（truncateSessionMemoryForCompact）
   → 每章节最多 2000 tokens，超出部分追加 "[... section truncated ...]"
   → 这是为了防止 session memory 文件本身太大

6. 组装压缩结果
   → [boundary marker]        ← 元数据标记
   → [session memory 内容]    ← role="user" 消息
   → [保留的最近消息]          ← 原样保留
   → [attachments/hooks]      ← 恢复 CLAUDE.md 等

7. 验证压缩后 token 数不超阈值
   → 如果压缩后仍超：返回 null，降级到 LLM Full Compact
```

**关键：`lastSummarizedMessageId` 的作用**

这个 UUID 是整个系统的"锚点"：
- 它标记了"session memory 文件内容覆盖到了对话的哪个位置"
- 压缩时，这个 UUID 之前的消息可以安全丢弃（因为已经被记录进文件了）
- 这个 UUID 之后的消息必须保留（还没被记录，丢了就真丢了）

如果最后一轮有工具调用，`updateLastSummarizedMessageIdIfSafe` 会跳过更新（等到工具链结束），防止"锚点"指向一个中间状态。

### 压缩时如何使用 Session Memory

当需要 Full Compact 时，如果已有 Session Memory，可以**跳过 LLM 调用**，直接利用已有摘要：

```typescript
// sessionMemoryCompact.ts
async function trySessionMemoryCompaction(
    messages: Message[],
    agentId?: AgentId,
    autoCompactThreshold?: number,
): Promise<CompactionResult | null> {
    // 等待可能正在进行的提取完成
    await waitForSessionMemoryExtraction();

    const sessionMemory = await getSessionMemoryContent();
    if (!sessionMemory || await isSessionMemoryEmpty(sessionMemory)) return null;

    // 找到 lastSummarizedMessageId：记忆文件最后一次覆盖到的消息 UUID
    const lastSummarizedIndex = messages.findIndex(
        msg => msg.uuid === lastSummarizedMessageId
    );
    if (lastSummarizedIndex === -1) return null;

    // 计算保留窗口：在 lastSummarizedIndex 之后，向前扩展到满足最小 token 要求
    const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex);
    const messagesToKeep = messages
        .slice(startIndex)
        .filter(m => !isCompactBoundaryMessage(m));  // 过滤边界标记消息

    // 构建压缩结果（不需要 LLM 调用）
    const compactionResult = createCompactionResultFromSessionMemory(
        messages,
        sessionMemory,   // 直接使用已有记忆
        messagesToKeep,
        hookResults,
        transcriptPath,
        agentId,
    );

    // 验证压缩后不超过触发阈值（否则这次压缩没意义）
    const postCompactTokenCount = estimateMessageTokens(
        buildPostCompactMessages(compactionResult)
    );
    if (autoCompactThreshold !== undefined
        && postCompactTokenCount >= autoCompactThreshold) {
        return null;  // 压缩后仍超标，放弃，走 LLM Full Compact
    }

    return { ...compactionResult, postCompactTokenCount };
}
```

**保留窗口的计算（calculateMessagesToKeepIndex）：**

```typescript
// 从 lastSummarizedIndex+1 开始，向前扩展，直到满足：
//   totalTokens ∈ [minTokens, maxTokens]  AND  textBlockMessages >= minTextBlockMessages
const config = {
    minTokens: 10_000,        // 保留至少 10K token 的上下文
    maxTokens: 40_000,        // 但不超过 40K（防止保留太多，压缩没效果）
    minTextBlockMessages: 5,  // 至少保留 5 条含文本的消息
};
```

### 摘要消息结构

压缩后插入的摘要消息（role="user"）：

```
This session is being continued from a previous conversation that has been compacted.

Summary:
[Session Memory 文件内容，按章节截断至每章 2000 tokens]

If you need specific details that aren't captured in this summary, you can
read the full transcript at: /path/to/.claude/projects/xxx/transcript.json

Recent messages are preserved verbatim below.
```

---

## 二、OpenClaw：Memory Flush 系统

### 设计理念："先写后压"

OpenClaw 的策略与 Claude Code 相反：**在压缩之前强制将记忆写入磁盘**。理由是：
- 压缩会丢失信息，因此在压缩前必须确保重要信息已持久化
- 压缩失败时，已 flush 的记忆不会丢失

### 触发条件

```typescript
// memory-flush.ts
function shouldRunMemoryFlush(params: {
    entry?: SessionEntry;
    tokenCount?: number;
    contextWindowTokens: number;
    reserveTokensFloor: number;    // 通常 = compaction.reserveTokens
    softThresholdTokens: number;   // 默认 4,000（softThreshold 之前触发 flush）
}): boolean {
    const threshold = params.contextWindowTokens
                    - params.reserveTokensFloor
                    - params.softThresholdTokens;

    // 当前 token 数必须超过阈值
    if (currentTokens < threshold) return false;

    // 同一 compaction 周期只 flush 一次（用 compactionCount 追踪）
    if (hasAlreadyFlushedForCurrentCompaction(entry)) return false;

    return true;
}

function hasAlreadyFlushedForCurrentCompaction(entry: SessionEntry): boolean {
    const compactionCount = entry.compactionCount ?? 0;
    const lastFlushAt = entry.memoryFlushCompactionCount;
    // 如果上次 flush 的 compaction 序号 === 当前序号，则已 flush
    return typeof lastFlushAt === 'number' && lastFlushAt === compactionCount;
}
```

### Context Hash 去重

为防止重复 flush（同一 context tail 多次触发），使用 SHA-256 hash：

```typescript
// memory-flush.ts
function computeContextHash(
    messages: Array<{ role?: string; content?: unknown }>
): string {
    // 取最近 3 条 user/assistant 消息
    const userAssistant = messages.filter(
        m => m.role === 'user' || m.role === 'assistant'
    );
    const tail = userAssistant.slice(-3);

    const payload = `${messages.length}:${tail.map((m, i) =>
        `[${i}:${m.role ?? ""}]${
            typeof m.content === 'string'
                ? m.content
                : JSON.stringify(m.content ?? "")
        }`
    ).join('\x00')}`;

    return crypto.createHash('sha256').update(payload).digest('hex').slice(0, 16);
}
```

### Memory Flush 执行过程

Memory Flush 实质上是**运行一个隐藏的 agent turn**：

```typescript
// agent-runner-memory.ts — runMemoryFlushIfNeeded()
async function runMemoryFlushIfNeeded(session, memoryDeps) {
    if (!shouldRunMemoryFlush({
        entry: session.entry,
        contextWindowTokens: session.contextWindowTokens,
        reserveTokensFloor: session.reserveTokensFloor,
        softThresholdTokens: memoryConfig.softThresholdTokens ?? 4_000,
    })) return;

    // 检查 context hash，相同则跳过（context 没变化）
    const currentHash = computeContextHash(session.messages);
    if (currentHash === session.entry.memoryFlushContextHash) return;

    // 执行记忆提取：运行一个不对用户可见的 agent turn
    const flushResult = await runHiddenAgentTurn({
        session,
        memoryFlushWritePath: session.config.memoryFlushWritePath,  // 记忆文件路径
        prompt: session.config.memoryFlushPrompt,                   // 提取指令
        extraSystemPrompt: [
            flushInstructions,               // 写入格式要求
            postCompactionRefreshPrompt,     // 压缩后恢复状态的提示
        ].join('\n'),
        silentExpected: true,   // 不向用户显示
        trigger: "memory",      // 特殊触发类型
    });

    // 更新 session 状态，标记已 flush
    await session.updateEntry({
        memoryFlushAt: Date.now(),
        memoryFlushContextHash: currentHash,
        memoryFlushCompactionCount: session.entry.compactionCount ?? 0,
    });
}
```

### Session 持久化（sessions.json + transcript.jsonl）

OpenClaw 用两个文件分别存储不同时效的数据：

```typescript
// sessions.json（小文件，频繁更新）
{
    "sessionId": "abc123",           // 当前 transcript 文件名
    "totalTokens": 45000,            // 累计 token 消耗
    "contextTokens": 38000,          // 当前 context 占用
    "compactionCount": 3,            // 压缩次数（用于 memoryFlush 去重）
    "memoryFlushAt": 1713187200000,  // 上次 flush 时间戳
    "memoryFlushContextHash": "a1b2c3d4e5f6a7b8",  // 上次 flush 时的 context hash
    "memoryFlushCompactionCount": 2  // 上次 flush 时的 compaction 序号
}

// transcript.jsonl（append-only，每条一行）
{"type":"message","id":"msg_001","role":"user","content":"..."}
{"type":"message","id":"msg_002","role":"assistant","content":"..."}
{"type":"compaction","id":"cmp_001","firstKeptEntryId":"msg_050","tokensBefore":180000}
{"type":"custom","id":"cus_001","data":{...}}  // 扩展状态（不进 model context）
```

**恢复机制**：
```typescript
// session.buildSessionContext()
function buildSessionContext(branch: SessionEntry[]): AgentMessage[] {
    const latestCompactionIdx = findLastIndex(branch, e => e.type === 'compaction');

    if (latestCompactionIdx === -1) {
        // 没有压缩历史，直接返回所有消息
        return branch.filter(e => e.type === 'message').map(toAgentMessage);
    }

    const compactionEntry = branch[latestCompactionIdx];
    const { firstKeptEntryId } = compactionEntry;

    // 找到 firstKeptEntryId 之后的消息（未被摘要的部分）
    const keptMessages: AgentMessage[] = [];
    let afterFirstKept = false;
    for (const entry of branch) {
        if (entry.id === firstKeptEntryId) afterFirstKept = true;
        if (afterFirstKept && entry.type === 'message') {
            keptMessages.push(toAgentMessage(entry));
        }
    }

    // 在 keptMessages 前插入压缩摘要
    return [
        { role: 'user', content: compactionEntry.summary },
        ...keptMessages,
    ];
}
```

### SQLite 记忆存储

跨会话的持久记忆存储在 `~/.openclaw/memory/<agentId>.sqlite`：

```typescript
// memory-search.ts 配置
const memoryConfig = {
    storage: `${homeDir}/.openclaw/memory/${agentId}.sqlite`,

    // 支持两种搜索来源
    sources: ["memory", "sessions"],
    // "memory"   — 只搜索 memory 目录下的 .md 文件
    // "sessions" — 还可搜索历史 session transcripts（跨会话检索）

    chunking: {
        defaultChunkTokens: 400,   // 每块约 400 token
        overlapTokens: 80,         // 80 token 重叠，避免切断语义单元
        ftsTokenizer: "unicode61", // 全文索引 tokenizer
    },

    search: {
        type: "hybrid",           // 向量 + BM25 全文检索
        vectorWeight: 0.7,        // 向量相似度权重
        textWeight: 0.3,          // 全文匹配权重
        candidateMultiplier: 4,   // 先取 maxResults*4 个候选，再 rerank
        mmr: {
            enabled: false,        // MMR 去重（默认关闭，可配置开启）
            lambda: 0.7,           // MMR 中相关性 vs 多样性的权衡
        },
        temporalDecay: {
            enabled: false,        // 时序衰减（默认关闭）
            halfLifeDays: 30,      // 30 天半衰期
        },
        maxResults: 6,
        minScore: 0.35,
    },
};
```

---

## 三、OpenHarness：文件系统 + 快照双轨

### 结构化记忆文件（Project Memory）

```
project-root/
  MEMORY.md                ← 索引（入口，按行列举所有记忆条目）
  memory/
    feedback_testing.md   ← 具体记忆条目（含 YAML frontmatter）
    user_role.md
    project_architecture.md
```

**写入记忆：**

```python
# memory/manager.py
def add_memory_entry(cwd: str | Path, title: str, content: str) -> Path:
    memory_dir = get_project_memory_dir(cwd)
    # slug 化文件名（特殊字符替换为下划线）
    slug = re.sub(r"[^a-zA-Z0-9]+", "_", title.strip().lower()).strip("_") or "memory"
    path = memory_dir / f"{slug}.md"

    with exclusive_file_lock(_memory_lock_path(cwd)):  # 防并发写入
        atomic_write_text(path, content.strip() + "\n")

        # 更新 MEMORY.md 索引
        entrypoint = get_memory_entrypoint(cwd)
        existing = entrypoint.read_text() if entrypoint.exists() else "# Memory Index\n"
        if path.name not in existing:  # 防重复
            existing = existing.rstrip() + f"\n- [{title}]({path.name})\n"
            atomic_write_text(entrypoint, existing)

    return path
```

**MEMORY.md 加载（注入 system prompt）：**

```python
# memory/memdir.py
def load_memory_prompt(cwd: str | Path, *, max_entrypoint_lines: int = 200) -> str | None:
    memory_dir = get_project_memory_dir(cwd)
    entrypoint = get_memory_entrypoint(cwd)
    lines = [
        "# Memory",
        f"- Persistent memory directory: {memory_dir}",
        "- Use this directory to store durable user or project context.",
        "- Prefer concise topic files plus an index entry in MEMORY.md.",
    ]

    if entrypoint.exists():
        # 最多读取 200 行，防止 MEMORY.md 过大
        content_lines = entrypoint.read_text().splitlines()[:max_entrypoint_lines]
        lines.extend(["", "## MEMORY.md", "```md", *content_lines, "```"])
    else:
        lines.extend(["", "## MEMORY.md", "(not created yet)"])

    return "\n".join(lines)
```

注入到 system prompt 后的格式：
```
# Memory
- Persistent memory directory: /path/to/project/memory
- Use this directory to store durable user or project context.
- Prefer concise topic files plus an index entry in MEMORY.md.

## MEMORY.md
```md
# Memory Index

- [不要 mock 数据库](feedback_testing.md) — 集成测试必须连真实 DB
- [用户角色](user_role.md) — data scientist，关注 observability
```
```

### Session 快照（ohmo 会话工具）

ohmo 把完整会话存为 JSON，支持恢复历史会话：

```python
# session_storage.py
def save_session_snapshot(
    *,
    cwd: str | Path,
    model: str,
    system_prompt: str,
    messages: list[ConversationMessage],
    usage: UsageSnapshot,
    session_id: str | None = None,
    session_key: str | None = None,
) -> Path:
    session_dir = get_session_dir(workspace)
    sid = session_id or uuid4().hex[:12]

    payload = {
        "app": "ohmo",
        "session_id": sid,
        "session_key": session_key,
        "cwd": str(Path(cwd).resolve()),
        "model": model,
        "system_prompt": system_prompt,  # 含记忆注入后的完整 system prompt
        "messages": [msg.model_dump(mode="json") for msg in messages],  # 完整消息列表
        "usage": usage.model_dump(),
        "summary": next(  # 快速预览：第一条用户消息的前 80 chars
            (m.text.strip()[:80] for m in messages if m.role == "user" and m.text.strip()),
            ""
        ),
        "message_count": len(messages),
        "created_at": time.time(),
    }

    # 三份存储
    atomic_write_text(session_dir / "latest.json",              json.dumps(payload))  # 快速加载最新
    atomic_write_text(session_dir / f"session-{sid}.json",      json.dumps(payload))  # 历史存档
    if session_key:
        atomic_write_text(_session_key_latest_path(workspace, session_key),           # 按 key 检索
                          json.dumps(payload))

    return session_dir / "latest.json"

def load_latest(workspace=None) -> dict | None:
    path = get_session_dir(workspace) / "latest.json"
    if not path.exists(): return None
    return json.loads(path.read_text(encoding="utf-8"))
```

**恢复会话：** 从 `latest.json` 读取 `messages` 列表，反序列化为 `ConversationMessage` 对象，直接恢复到压缩后的状态继续对话。

---

## Claude Code 的 MEMORY.md 索引截断保护

```typescript
// memdir.ts
const MAX_ENTRYPOINT_LINES = 200;
const MAX_ENTRYPOINT_BYTES = 25_000;  // 25KB

function truncateEntrypointContent(raw: string): EntrypointTruncation {
    const trimmed = raw.trim();
    const lines = trimmed.split('\n');

    const wasLineTruncated = lines.length > MAX_ENTRYPOINT_LINES;
    const wasByteTruncated = trimmed.length > MAX_ENTRYPOINT_BYTES;

    if (!wasLineTruncated && !wasByteTruncated) {
        return { content: trimmed, wasLineTruncated: false, wasByteTruncated: false };
    }

    let truncated = wasLineTruncated
        ? lines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
        : trimmed;

    if (truncated.length > MAX_ENTRYPOINT_BYTES) {
        const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES);
        truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES);
    }

    const reason = wasLineTruncated
        ? `${lines.length} lines (limit: ${MAX_ENTRYPOINT_LINES})`
        : `${formatFileSize(trimmed.length)} (limit: ${formatFileSize(MAX_ENTRYPOINT_BYTES)})`;

    return {
        content: truncated + `\n\n> WARNING: MEMORY.md is ${reason}. ` +
            `Only part was loaded. Keep index entries to one line under ~200 chars; ` +
            `move detail into topic files.`,
        wasLineTruncated,
        wasByteTruncated,
    };
}
```

---

## 三系统对比

| 特性 | Claude Code | OpenClaw | OpenHarness |
|------|------------|---------|-------------|
| 会话内记忆 | Forked agent 异步提取 | 压缩前 Memory Flush | Session Memory 确定性摘要 |
| 提取时机 | 每轮 query 后（后台） | 压缩触发前（前置） | 手动 / 自动触发 |
| 跨会话存储 | Markdown 文件 | SQLite 数据库 | Markdown 文件 + JSON 快照 |
| 并发保护 | `sequential()` wrapper | `compactionCount` 计数器 | `exclusive_file_lock` |
| 索引大小限制 | 200 行 / 25KB | 无显式限制 | 200 行（可配置） |
| 搜索能力 | LLM 语义选择 | 向量 + 全文混合 | 关键词权重打分 |
| 时序衰减 | 无（按 mtime 排序） | 可选 30 天半衰期 | 按 mtime 排序 |
| 跨 session 搜索 | 无 | 可搜索历史 transcript | 无 |
| 恢复机制 | 从 MEMORY.md 注入 | 从 transcript.jsonl 重建 | 从 latest.json 反序列化 |
