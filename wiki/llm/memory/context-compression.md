---
title: Context Compression
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - claude-code/src/services/compact/compact.ts
  - claude-code/src/services/compact/prompt.ts
  - claude-code/src/services/compact/autoCompact.ts
  - claude-code/src/services/compact/grouping.ts
  - claude-code/src/utils/tokens.ts
  - openclaw/src/agents/pi-embedded-runner/compact.ts
  - openclaw/src/agents/pi-embedded-runner/tool-result-truncation.ts
  - openclaw/src/agents/context-window-guard.ts
  - OpenHarness/src/openharness/services/compact/__init__.py
tags:
  - context-window
  - compression
  - compact
  - token
  - prompt-engineering
---

# Context Compression

## 问题背景

当对话历史积累到接近 context window 上限时，系统必须在**丢失信息**和**无法继续**之间做出权衡。

好的压缩系统需要满足：
- **渐进式**：从廉价到昂贵，能止则止
- **安全**：不破坏 API message 的结构约束（user/assistant 必须交替，tool_use/tool_result 必须配对）
- **可重试**：压缩请求本身也可能触发 prompt-too-long 错误

---

## 触发时机

### Token 计数策略

精确的 token 数只有在 API 响应中才知道（从 `usage` 字段）。因此三个系统都用**混合估算**：

```typescript
// Claude Code: tokens.ts
// 策略：找最近一条有真实 usage 数据的 assistant message，
// 真实计数 + 该消息之后的粗估，比全程粗估准确得多
function tokenCountWithEstimation(messages: readonly Message[]): number {
    let i = messages.length - 1;
    while (i >= 0) {
        const usage = getTokenUsage(messages[i]);
        if (usage) {
            // 同一 API response 的多个 chunk 有相同 message.id，
            // 往前找到最早的同 id 消息作为锚点
            const responseId = getAssistantMessageId(messages[i]);
            if (responseId) {
                let j = i - 1;
                while (j >= 0) {
                    const priorId = getAssistantMessageId(messages[j]);
                    if (priorId === responseId) { i = j; }
                    else if (priorId !== undefined) { break; }
                    j--;
                }
            }
            return getTokenCountFromUsage(usage)
                 + roughTokenCountEstimationForMessages(messages.slice(i + 1));
        }
        i--;
    }
    return roughTokenCountEstimationForMessages(messages);  // 全程粗估兜底
}

// 粗估：4 bytes per token（经验值）
function roughTokenCountEstimation(content: string): number {
    return Math.round(content.length / 4);
}
```

```python
# OpenHarness: compact/__init__.py
# 加 33% 保守 padding，宁可早触发
TOKEN_ESTIMATION_PADDING = 4 / 3

def estimate_tokens(text: str) -> int:
    return int(len(text) / 4 * TOKEN_ESTIMATION_PADDING)
```

### 阈值计算（三系统一致）

```
Model Context Window        200,000 tokens
  - Reserved for Output      20,000 tokens  (为模型回复预留，取 max_output 和 20K 的较小值)
  - Autocompact Buffer       13,000 tokens  (安全边距，防止因估算误差撞到硬限制)
  ──────────────────────────────────────
  Effective Threshold    ≈ 167,000 tokens  ← 触发压缩
```

```typescript
// Claude Code: autoCompact.ts
function getAutoCompactThreshold(model: string): number {
    const reservedForSummary = Math.min(
        getMaxOutputTokensForModel(model),
        MAX_OUTPUT_TOKENS_FOR_SUMMARY,  // 20,000
    );
    let contextWindow = getContextWindowForModel(model, getSdkBetas());

    // 允许通过环境变量缩小窗口（便于测试）
    const envOverride = process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW;
    if (envOverride) contextWindow = Math.min(contextWindow, parseInt(envOverride));

    const effectiveWindow = contextWindow - reservedForSummary;
    return effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS;  // 13,000
}
```

---

## 三系统的压缩流水线

### OpenHarness：最明确的四级流水线

```python
# query_engine.py 中每轮 query 前调用
async def auto_compact_if_needed(
    messages: list[ConversationMessage],
    *,
    api_client, model, system_prompt,
    state: AutoCompactState,
    preserve_recent: int = 6,
    context_window_tokens: int | None = None,
    auto_compact_threshold_tokens: int | None = None,
) -> tuple[list[ConversationMessage], bool]:

    if not force and not should_autocompact(messages, model, state, ...):
        return messages, False

    # ── Level 1: 微压缩 ───────────────────────────────────────
    messages, tokens_freed = microcompact_messages(messages)
    if tokens_freed > 0 and not should_autocompact(messages, model, state, ...):
        log.info("Microcompact freed ~%d tokens, no further compact needed", tokens_freed)
        return messages, True

    # ── Level 2: 上下文折叠 ────────────────────────────────────
    collapsed = try_context_collapse(messages, preserve_recent=preserve_recent)
    if collapsed is not None:
        messages = collapsed
        if not force and not should_autocompact(messages, model, state, ...):
            return messages, True

    # ── Level 3: Session Memory 确定性摘要 ────────────────────
    sm_result = try_session_memory_compaction(messages, preserve_recent=preserve_recent)
    if sm_result is not None:
        return build_post_compact_messages(sm_result), True

    # ── Level 4: LLM Full Compact（最贵最强）────────────────────
    result = await compact_conversation(
        messages,
        api_client=api_client,
        model=model,
        system_prompt=system_prompt,
        preserve_recent=preserve_recent,
        trigger="auto",
    )
    state.consecutive_failures = 0
    return build_post_compact_messages(result), True
```

### Claude Code：Session Memory 优先策略

```typescript
// autoCompact.ts — autoCompactIfNeeded()
async function autoCompactIfNeeded(
    messages: Message[],
    model: string,
    ...
): Promise<AutoCompactResult> {

    // 熔断器：连续失败超过 3 次则停止尝试，防止死循环
    if (state.consecutiveFailures >= MAX_CONSECUTIVE_FAILURES) {
        return { wasCompacted: false };
    }

    if (!await shouldAutoCompact(messages, model, querySource, snipTokensFreed)) {
        return { wasCompacted: false };
    }

    // ── 优先：Session Memory 压缩（无需重新 LLM 调用）────────────
    const smResult = await trySessionMemoryCompaction(
        messages, agentId, autoCompactThreshold
    );
    if (smResult !== null) {
        const postCompactMessages = buildPostCompactMessages(smResult);
        await runPostCompactCleanup(...);
        markPostCompaction();
        return { wasCompacted: true, messages: postCompactMessages };
    }

    // ── 降级：LLM Full Compact ────────────────────────────────
    try {
        const result = await compactConversation({
            messages, model, isAutoCompact: true, ...
        });
        state.consecutiveFailures = 0;
        return { wasCompacted: true, messages: buildPostCompactMessages(result) };
    } catch (err) {
        state.consecutiveFailures++;
        return { wasCompacted: false, error: err };
    }
}
```

### OpenClaw：工具截断 + LLM Compact 两级

```typescript
// compact.ts — 核心流程
// 1. sanitize: 清理格式异常的消息
const sanitized = sanitizeSessionHistory(sessionHistory);
// 2. validate: 检查重放轮次是否合法
validateReplayTurns(sanitized);
// 3. limit: 限制历史轮数（防止过深递归）
const limited = limitHistoryTurns(sanitized, params.maxHistoryTurns);
// 4. repair: 修复 tool_use/tool_result 配对断裂
const repaired = sanitizeToolUseResultPairing(limited);
// 5. 调用底层 pi-library 做摘要
const compactResult = await session.compact(params.customInstructions);
```

工具截断（在发送给模型前预处理）：
```typescript
// tool-result-truncation.ts — 两阶段截断
function buildToolResultReplacementPlan(params: {
    branch: ToolResultBranchEntry[];
    maxChars: number;          // 单条上限，默认 40,000 chars
    aggregateBudgetChars: number;  // 总量上限 = contextWindow * 0.3 * 4
}) {
    // Phase 1: 处理单条超大的 tool result
    const oversizedReplacements = buildOversizedToolResultReplacements({
        branch: params.branch,
        maxChars: params.maxChars,
        minKeepChars: 2_000,
    });

    // Phase 2: 处理总量超标（即使每条都不超标，加起来可能超）
    const trimmedBranch = applyToolResultReplacementsToBranch(
        params.branch, oversizedReplacements
    );
    const aggregateReplacements = buildAggregateToolResultReplacements({
        branch: trimmedBranch,
        aggregateBudgetChars: params.aggregateBudgetChars,
        minKeepChars: 2_000,
    });

    return {
        replacements: [...oversizedReplacements, ...aggregateReplacements],
    };
}
```

---

## 各级压缩详解

### 微压缩（Microcompact）— 无 LLM

清理旧的 tool results，用占位符替换：

```python
# OpenHarness: compact/__init__.py
COMPACTABLE_TOOLS = frozenset({
    "read_file", "bash", "grep", "glob",
    "web_search", "web_fetch", "edit_file", "write_file",
})
DEFAULT_KEEP_RECENT = 5  # 保留最近 5 条 tool result

def microcompact_messages(
    messages: list[ConversationMessage],
    keep_recent: int = DEFAULT_KEEP_RECENT,
) -> tuple[list[ConversationMessage], int]:
    keep_recent = max(1, keep_recent)  # 至少保留 1 条
    all_ids = _collect_compactable_tool_ids(messages)

    if len(all_ids) <= keep_recent:
        return messages, 0  # 不需要清理

    keep_set  = set(all_ids[-keep_recent:])  # 保留最新的
    clear_set = set(all_ids) - keep_set      # 清理其余的

    tokens_saved = 0
    for msg in messages:
        if msg.role != "user":
            continue
        new_content = []
        for block in msg.content:
            if (isinstance(block, ToolResultBlock)
                    and block.tool_use_id in clear_set
                    and block.content != TIME_BASED_MC_CLEARED_MESSAGE):
                tokens_saved += estimate_tokens(block.content)
                new_content.append(ToolResultBlock(
                    tool_use_id=block.tool_use_id,
                    content=TIME_BASED_MC_CLEARED_MESSAGE,  # "[cleared by microcompact]"
                    is_error=block.is_error,
                ))
            else:
                new_content.append(block)
        msg.content = new_content

    return messages, tokens_saved
```

### 上下文折叠（Context Collapse）— 无 LLM

对老消息中的长文本做**头尾截断**：

```python
# OpenHarness: compact/__init__.py
CONTEXT_COLLAPSE_TEXT_CHAR_LIMIT = 2_400  # 超过此长度才折叠
CONTEXT_COLLAPSE_HEAD_CHARS = 900
CONTEXT_COLLAPSE_TAIL_CHARS = 500

def _collapse_text(text: str) -> str:
    if len(text) <= CONTEXT_COLLAPSE_TEXT_CHAR_LIMIT:
        return text
    omitted = len(text) - CONTEXT_COLLAPSE_HEAD_CHARS - CONTEXT_COLLAPSE_TAIL_CHARS
    head = text[:CONTEXT_COLLAPSE_HEAD_CHARS].rstrip()
    tail = text[-CONTEXT_COLLAPSE_TAIL_CHARS:].lstrip()
    return f"{head}\n...[collapsed {omitted} chars]...\n{tail}"

def try_context_collapse(
    messages: list[ConversationMessage],
    preserve_recent: int,
) -> list[ConversationMessage] | None:
    if len(messages) <= preserve_recent + 2:
        return None

    older = messages[:-preserve_recent]
    newer = messages[-preserve_recent:]
    changed = False

    collapsed_older = []
    for message in older:
        new_blocks = []
        for block in message.content:
            if isinstance(block, TextBlock):
                collapsed = _collapse_text(block.text)
                changed = changed or (collapsed != block.text)
                new_blocks.append(TextBlock(text=collapsed))
            else:
                new_blocks.append(block)
        collapsed_older.append(ConversationMessage(
            role=message.role, content=new_blocks
        ))

    if not changed:
        return None

    result = [*collapsed_older, *newer]
    # 安全检查：必须确实减少了 token
    if estimate_message_tokens(result) >= estimate_message_tokens(messages):
        return None
    return result
```

OpenClaw 对 tool result 的截断有更精细的头尾策略，会检测尾部是否含有重要内容：

```typescript
// tool-result-truncation.ts
const MIDDLE_OMISSION_MARKER = "\n\n⚠️ [... middle content omitted ...]\n\n";
const MIN_KEEP_CHARS = 2_000;

// 检测尾部是否含有重要信息
function hasImportantTail(text: string): boolean {
    const tail = text.slice(-2000).toLowerCase();
    return (
        /\b(error|exception|failed|fatal|traceback|panic|stack trace)\b/.test(tail) ||
        /\}\s*$/.test(tail.trim()) ||   // JSON 结构闭合
        /\b(total|summary|result|complete|finished|done)\b/.test(tail)
    );
}

function truncateToolResultText(text: string, budget: number): string {
    if (text.length <= budget) return text;

    if (hasImportantTail(text) && budget > MIN_KEEP_CHARS * 2) {
        // 有重要尾部：3:7 分配，tail 占 30%（最多 4000 chars）
        const tailBudget = Math.min(Math.floor(budget * 0.3), 4_000);
        const headBudget = budget - tailBudget - MIDDLE_OMISSION_MARKER.length;
        const headCut = findCleanNewlineBoundary(text, headBudget);
        const tailStart = text.length - tailBudget;
        return text.slice(0, headCut) + MIDDLE_OMISSION_MARKER + text.slice(tailStart);
    } else {
        // 无重要尾部：只保留开头
        const cut = findCleanNewlineBoundary(text, budget);
        return text.slice(0, cut) + formatContextLimitTruncationNotice(text.length - cut);
    }
}
```

### Session Memory 确定性摘要 — 无 LLM

**这一层解决的核心问题**：LLM Full Compact 成本高、有延迟，但有时候只需要让消息列表"变短"，而不需要真正理解内容。确定性摘要就是这个"廉价替代品"——不调用 LLM，纯粹用规则把老消息压缩成一段结构化文本。

**它做的事**：把需要删除的老消息逐条转换为一行摘要（角色 + 前 160 字符，或工具名列表），拼成一个新的 user message 插在对话头部，替换掉原来的一大片消息。效果类似"把前 N 条消息折叠成一段会议纪要"。

输出样式：
```
Session memory summary from earlier in this conversation:
user: 我想给这个项目添加一个登录功能，用 JWT
assistant: tool calls -> read_file, bash
user: 不对，我要用 session cookie 而不是 JWT
assistant: 好的，我来修改认证逻辑
assistant: tool results returned
user: 还需要加 CSRF 保护
... earlier context condensed ...
```

**和 LLM Full Compact 的本质区别**：

| | 确定性摘要 | LLM Full Compact |
|--|-----------|----------------|
| 调用 LLM | 否 | 是 |
| 信息密度 | 低（只有摘要行） | 高（完整 9 章摘要） |
| 适合场景 | 消息多但内容简单 | 深度技术对话、需要还原细节 |
| 失败可能性 | 几乎没有 | 可能超时、PTL 错误 |
| 执行时间 | 毫秒级 | 数秒至数十秒 |

**什么时候有效**：对话中有大量短消息、工具调用往返频繁，但每条消息本身信息量不大。这种场景下确定性摘要能把几十条消息压成几百字，效果好。

**什么时候失效**：如果消息数量不够多（`len(messages) <= preserve_recent + 4`），或者压缩后 token 数不降反升（比如每条消息都很短，摘要行反而增加了 overhead），则返回 `None`，交由 LLM Full Compact 处理。

将老消息压缩为结构化摘要，**不调用 LLM**：

```python
# OpenHarness: compact/__init__.py
SESSION_MEMORY_KEEP_RECENT = 12   # 保留最近 12 条完整消息
SESSION_MEMORY_MAX_LINES   = 48   # 摘要最多 48 行
SESSION_MEMORY_MAX_CHARS   = 4_000

# 每条消息的摘要生成规则
def _summarize_message_for_memory(message: ConversationMessage) -> str:
    text = " ".join(message.text.split())
    if text:
        return f"{message.role}: {text[:160]}"  # 文本消息：截取前 160 chars

    tool_uses = [block.name for block in message.tool_uses]
    if tool_uses:
        return f"{message.role}: tool calls -> {', '.join(tool_uses[:4])}"  # 工具调用：列出工具名

    if any(isinstance(block, ToolResultBlock) for block in message.content):
        return f"{message.role}: tool results returned"  # 工具结果：仅标注

    return f"{message.role}: [non-text content]"

def _build_session_memory_message(
    messages: list[ConversationMessage]
) -> ConversationMessage | None:
    lines = []
    total_chars = 0

    for message in messages:
        line = _summarize_message_for_memory(message)
        if not line:
            continue
        if lines and (len(lines) >= SESSION_MEMORY_MAX_LINES
                      or total_chars + len(line) >= SESSION_MEMORY_MAX_CHARS):
            lines.append("... earlier context condensed ...")
            break
        lines.append(line)
        total_chars += len(line) + 1

    if not lines:
        return None

    return ConversationMessage.from_user_text(
        "Session memory summary from earlier in this conversation:\n" + "\n".join(lines)
    )
```

---

## LLM Full Compact：Prompt 完整解析

这是三个系统最核心、也最相似的部分。以下展示完整的 prompt 结构。

### Prompt 组成（三系统共同模式）

```
[1] NO_TOOLS_PREAMBLE     ← 强制禁用工具调用
[2] BASE_COMPACT_PROMPT   ← 结构化摘要指令（含 9 个输出章节）
[3] 自定义指令（可选）     ← 用户/系统注入的额外要求
[4] NO_TOOLS_TRAILER      ← 末尾重复禁用工具提醒
```

### NO_TOOLS_PREAMBLE（完整文本）

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

**为什么需要这个？** 压缩请求的上下文中包含大量工具调用记录，模型容易"惯性"地想继续调用工具。这个前置警告是防止模型"走神"的关键。

### BASE_COMPACT_PROMPT（完整结构）

```
Your task is to create a detailed summary of the conversation so far.
This summary will replace the earlier messages, so it must capture all important information.

First, draft your analysis inside <analysis> tags. Walk through the conversation
chronologically and extract:
- Every user request and intent (explicit and implicit)
- The approach taken and technical decisions made
- Specific code, files, and configurations discussed (with paths and line numbers where available)
- All errors encountered and how they were fixed
- Any user feedback or corrections

Then, produce a structured summary inside <summary> tags with these sections:

1. **Primary Request and Intent**
   All user requests in full detail, including nuances and constraints.

2. **Key Technical Concepts**
   Technologies, frameworks, patterns, and conventions discussed.

3. **Files and Code Sections**
   Every file examined or modified, with specific code snippets and line numbers.

4. **Errors and Fixes**
   Every error encountered, its cause, and how it was resolved.

5. **Problem Solving**
   Problems solved and approaches that worked vs. didn't work.

6. **All User Messages**
   Non-tool-result user messages (preserve exact wording for intent and feedback).

7. **Pending Tasks**
   Explicitly requested work that hasn't been completed yet.

8. **Current Work**
   Detailed description of the last task being worked on before compaction.
   Include file names, code snippets, exact state of progress.

9. **Optional Next Step**
   The single most logical next step, directly aligned with the user's most recent request.
   Include verbatim quotes from recent messages showing where work left off.
```

**设计要点：**
- `<analysis>` 是**思考草稿区**：模型先推理再输出，但 `<analysis>` 内容会在后处理时被剥离，不占用压缩后的 context
- Section 6（All User Messages）是防止**意图漂移**的关键：原始用户话语保留，不经过模型重新诠释
- Section 9 只写**一个**下一步，避免模型生成冗余建议

### 输出后处理

```python
# OpenHarness: compact/__init__.py
def format_compact_summary(raw_summary: str) -> str:
    # 1. 剥离 <analysis> 草稿区（可能很长）
    text = re.sub(r"<analysis>[\s\S]*?</analysis>", "", raw_summary)

    # 2. 将 <summary>...</summary> 标签替换为可读标题
    m = re.search(r"<summary>([\s\S]*?)</summary>", text)
    if m:
        text = text.replace(m.group(0), f"Summary:\n{m.group(1).strip()}")

    # 3. 清理多余空行
    text = re.sub(r"\n\n+", "\n\n", text)
    return text.strip()
```

```typescript
// Claude Code: prompt.ts — formatCompactSummary()
// 逻辑相同：剥离 <analysis>，提取 <summary> 内容
function formatCompactSummary(rawSummary: string): string {
    let text = rawSummary.replace(/<analysis>[\s\S]*?<\/analysis>/g, '');
    const summaryMatch = text.match(/<summary>([\s\S]*?)<\/summary>/);
    if (summaryMatch) {
        text = text.replace(summaryMatch[0], `Summary:\n${summaryMatch[1].trim()}`);
    }
    return text.replace(/\n\n+/g, '\n\n').trim();
}
```

### Claude Code 的 PARTIAL Compact（增量摘要）

Claude Code 有完整版和**增量版**两种 prompt：

```
BASE_COMPACT_PROMPT       — 对全部历史做摘要（history 被完全替换）
PARTIAL_COMPACT_PROMPT    — 只摘要 [older] 部分，[newer] 原样保留
PARTIAL_COMPACT_UP_TO_PROMPT — 只摘要 [up to N 轮] 的部分，后续消息接在摘要后
```

增量版摘要时，prompt 末尾会追加：
```
Note: The most recent messages are being preserved verbatim and will follow
this summary. Focus your summary on the earlier portion of the conversation.
```

这让 Claude Code 可以**只压缩已确定不再需要完整保留的部分**，而不是每次都重新摘要整个历史。

### 压缩后消息的插入位置

```typescript
// Claude Code: compact.ts — buildPostCompactMessages()
// 最终消息列表顺序：
[
    boundaryMarker,      // SystemMessage: 标记压缩点，含 pre/post token 元数据
    ...summaryMessages,  // UserMessage: 摘要内容（role="user"，不是 assistant）
    ...messagesToKeep,   // 保留的最近 N 条消息（原样）
    ...attachments,      // 重注入的附件（如 CLAUDE.md，从 session hooks 恢复）
    ...hookResults,      // post-compact hooks 的结果（如工具状态恢复）
]
```

**摘要为什么是 `role="user"`？** 这是一个工程上的关键决定：
- 模型对 `user` 消息有更强的"遵从"倾向
- 作为 user message，它在语义上代表"这是你之前对话的背景"，而不是"模型说了什么"
- 保持 API 的 user/assistant 交替结构

OpenClaw 同样如此：
```typescript
// compaction.ts
const summaryMessages: AgentMessage[] = partialSummaries.map((summary) => ({
    role: "user",      // ← 摘要作为 user message 插入
    content: summary,  // ← 纯文本，无 XML 标签
    timestamp: Date.now(),
}));
```

---

## Prompt-Too-Long 重试

压缩请求本身也可能超出 context window。三个系统都有重试：

```typescript
// Claude Code: compact.ts
const MAX_PTL_RETRIES = 3;

function truncateHeadForPTLRetry(
    messages: Message[],
    ptlResponse: AssistantMessage,
): Message[] | null {
    // 按 API round-trip 分组（共享同一 message.id 的 streaming chunks 算一组）
    const groups = groupMessagesByApiRound(messages);
    if (groups.length < 2) return null;  // 不能再删了

    // 如果能获取到精确 token gap，按 gap 精确计算需要丢弃多少 groups
    const tokenGap = getPromptTooLongTokenGap(ptlResponse);
    let dropCount: number;
    if (tokenGap !== undefined) {
        let acc = 0;
        dropCount = 0;
        for (const g of groups) {
            acc += roughTokenCountEstimationForMessages(g);
            dropCount++;
            if (acc >= tokenGap) break;
        }
    } else {
        // 无法获取 gap：保守丢弃 20%
        dropCount = Math.max(1, Math.floor(groups.length * 0.2));
    }

    dropCount = Math.min(dropCount, groups.length - 1);  // 保留至少 1 个 group
    const sliced = groups.slice(dropCount).flat();

    // 修复 API 不变量：消息列表必须以 user message 开头
    if (sliced[0]?.type === 'assistant') {
        return [
            createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }),
            ...sliced,
        ];
    }
    return sliced;
}

// 主压缩循环
for (;;) {
    summaryResponse = await streamCompactSummary({ messages: messagesToSummarize, ... });
    summary = getAssistantMessageText(summaryResponse);

    if (!summary?.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)) break;  // 成功退出

    ptlAttempts++;
    const truncated = ptlAttempts <= MAX_PTL_RETRIES
        ? truncateHeadForPTLRetry(messagesToSummarize, summaryResponse)
        : null;

    if (!truncated) throw new Error(ERROR_MESSAGE_PROMPT_TOO_LONG);
    messagesToSummarize = truncated;
}
```

```python
# OpenHarness: compact/__init__.py — 相同逻辑，Python 实现
MAX_PTL_RETRIES = 3
MAX_COMPACT_STREAMING_RETRIES = 2

for attempt in range(1, MAX_COMPACT_STREAMING_RETRIES + 2):
    try:
        summary_text = await asyncio.wait_for(
            _collect_summary(retry_messages),
            timeout=COMPACT_TIMEOUT_SECONDS,  # 25 秒超时
        )
        break
    except Exception as exc:
        if _is_prompt_too_long_error(exc) and ptl_retries < MAX_PTL_RETRIES:
            truncated = truncate_head_for_ptl_retry(retry_messages[:-1])
            if truncated:
                ptl_retries += 1
                retry_messages = [*truncated, retry_messages[-1]]
                continue
        if attempt > MAX_COMPACT_STREAMING_RETRIES:
            raise
```

---

## 其他压缩机制

### Time-based Microcompact（Claude Code）

**机制**：除 token 阈值触发外，Claude Code 还有一个**基于时间间隔**的微压缩，专门处理"长时间暂停后继续"的场景。

思路：如果用户离开超过 N 分钟（默认 60 分钟），之前的 tool results 价值已经很低，可以主动清理，为后续对话腾空间。这是独立于 token 计数的触发路径。

```ts
// time-based microcompact 逻辑
function shouldRunTimedMicrocompact(messages, lastActivityTime):
    gapMinutes = (now - lastActivityTime) / 60_000
    if gapMinutes < GAP_THRESHOLD_MINUTES:  // 默认 60 分钟
        return false
    if not hasCompactableToolResults(messages):
        return false
    return true

// 触发时机：在每轮 query 开始前检查
// 效果：清理超过 60 分钟前的 tool results，保留最近 N 条
// 不同于 token 触发：即使 token 没超标也会执行
```

`timeBasedMCConfig.ts` 里的参数通过 GrowthBook feature flag 下发，可以远程调整间隔和保留条数。

---

### Token Warning 状态机（Claude Code）

**机制**：在 Auto-compact 触发前，用多级阈值给用户显示进度提示，让用户有机会手动干预。

```
Context Window 的五个阶段：

  0%              75%     85%      93%     100%
  ├───────────────┼───────┼────────┼───────┤
  │  正常区间      │ 黄色  │  红色  │ 自动  │ 阻塞
  │               │ 警告  │  警告  │ 压缩  │ 限制

对应常量（均相对于 effective window）：
  WARNING_THRESHOLD_BUFFER  = 20,000 tokens  → 黄色警告
  ERROR_THRESHOLD_BUFFER    = 20,000 tokens  → 红色警告（同值，但 UI 颜色不同）
  AUTOCOMPACT_BUFFER        = 13,000 tokens  → 触发 auto-compact
  MANUAL_COMPACT_BUFFER     = 3,000 tokens   → 阻塞（手动 /compact 也需要这点空间）
```

```ts
// calculateTokenWarningState 返回的状态字段
{
    percentLeft: number,             // 距阈值还剩多少 %（UI 显示）
    isAboveWarningThreshold: bool,   // 是否显示黄色警告
    isAboveErrorThreshold: bool,     // 是否显示红色警告
    isAboveAutoCompactThreshold: bool, // 是否触发 auto-compact
    isAtBlockingLimit: bool,         // 是否禁止继续（即使手动 /compact 也没空间了）
}

// UI 显示逻辑（TokenWarning.tsx）：
if autoCompactEnabled:
    显示 "{percentLeft}% until auto-compact"
else:
    显示 "Context low ({percentLeft}% remaining) · Run /compact to compact & continue"
```

这个状态机驱动 UI 的颜色和文案，同时也是 `shouldAutoCompact()` 的判断依据（`isAboveAutoCompactThreshold`）。

---

### Snip 机制（Claude Code，计划中）

**机制**：一种比 microcompact 更激进的内存截断，专门为**SDK 无 UI 的长时间运行场景**设计。

背景：REPL（终端）模式下需要保留完整历史用于 UI 滚动。但 SDK 模式（headless）下不需要 UI 历史，可以更激进地截断内存中的消息列表，防止内存泄漏。

```ts
// Snip 触发流程（QueryEngine.ts 中，通过 feature flag 注入）
当 模型返回一个 "snip boundary" 系统消息 时：
    snipCompactIfNeeded(mutableMessages, { force: true })
        → 直接截断内存中的消息列表
        → 不调用 LLM，不生成摘要
        → 替换 QueryEngine 内部的 mutableMessages

// 区别于 auto-compact：
// - auto-compact 是响应 token 阈值，生成摘要后替换
// - snip 是响应 boundary 信号，直接截断，无摘要
// - snip 只在 SDK 模式启用（HISTORY_SNIP feature flag）
```

目前 `snipCompact.ts` 和 `snipProjection.ts` 尚未实现，代码中已有框架（feature flag + 接口定义），是规划中的功能。

---

### OpenClaw Session Truncation + Orphan Re-parenting

**机制**：压缩完成后，对 `transcript.jsonl` 文件进行物理截断，防止文件无限增长。关键挑战是处理树形结构中的"孤儿"条目。

```ts
// 为什么需要 re-parenting？
// transcript.jsonl 存储的是有向树（支持分支/撤销），每个 entry 有 parentId
// 压缩删除了一批老消息后，某些条目的 parent 被删掉了 → 成为孤儿

// truncateSessionFile 算法：
function truncateSessionFile(allEntries, branch):
    // Step 1: 找到最新的 compaction entry，读取 firstKeptEntryId
    compactionEntry = findLatestCompaction(branch)
    firstKeptId = compactionEntry.firstKeptEntryId

    // Step 2: 确定哪些 entry 是"已被摘要"的
    summarizedIds = {}
    for entry in branch before compactionEntry:
        if entry.id == firstKeptId: break  // 这里之后的是未摘要的"尾部"
        summarizedIds.add(entry.id)

    // Step 3: 标记要删除的 entry（message 类型的才删，其他类型保留）
    toRemove = {}
    for entry in allEntries:
        if entry.id in summarizedIds and entry.type == "message":
            toRemove.add(entry.id)
        // 级联删除：label/branch_summary 指向被删 entry 的也删
        if entry.type == "label" and entry.targetId in toRemove:
            toRemove.add(entry.id)

    // Step 4: Re-parenting——找每个孤儿的最近存活祖先
    for entry in allEntries:
        if entry.id in toRemove: skip
        newParentId = entry.parentId
        while newParentId != null and newParentId in toRemove:
            newParentId = entryById[newParentId].parentId  // 往上找
        if newParentId != entry.parentId:
            entry.parentId = newParentId  // 接到最近存活的祖先上
        keep(entry)
```

Re-parenting 保证了：删除老消息后，树结构依然连通，分支历史不会断裂。

---

### OpenClaw 压缩安全超时

**机制**：压缩调用 LLM 生成摘要，理论上可能卡死。Safety timeout 在 15 分钟后强制终止。

```ts
// compactWithSafetyTimeout 伪代码
function compactWithSafetyTimeout(compactFn, timeoutMs=900_000):
    // 同时监听两个取消信号：
    //   1. 内部超时（15 分钟）
    //   2. 外部 AbortSignal（用户手动取消）
    result = await race(
        compactFn(),
        timeout(timeoutMs),      // 15 分钟
        externalAbort(),         // 用户取消
    )
    if timed_out:
        cancel()                 // 调用 onCancel hook（best-effort）
        throw TimeoutError
    return result

// 可通过配置覆盖：
//   agents.defaults.compaction.timeoutSeconds = 600  // 改为 10 分钟
// 上限：MAX_SAFE_TIMEOUT_MS（约 2,147 秒 ≈ 35 分钟）
```

超时后，当前这轮压缩失败，下一轮 query 会重新触发（OpenClaw 有 consecutive failure 计数器，连续失败超限则停止重试）。

---

### Prompt Cache 复用（Claude Code）

**机制**：LLM Full Compact 时，压缩请求复用主对话的 prompt cache，避免重复付 cache creation 的 token 成本。

背景：Claude API 的 prompt cache 按前缀缓存。压缩 fork 和主对话共享相同的 system prompt、tools 等前缀，理论上可以命中主对话建立的 cache。

```ts
// cache piggybacking 逻辑
if feature("tengu_compact_cache_prefix") enabled:
    // 用 runForkedAgent 而不是直接 API 调用
    // forked agent 携带 cacheSafeParams（包含主对话的 cache key 参数）
    result = await runForkedAgent({
        promptMessages: [summaryRequest],
        cacheSafeParams: {
            system,           // 必须与主对话完全一致
            tools,            // 必须与主对话完全一致
            model,            // 必须与主对话完全一致
            forkContextMessages: messagesToSummarize,
            thinkingConfig: { type: "disabled" },  // 注意：禁用 thinking（见下节）
        },
        skipCacheWrite: true,   // 只读 cache，不写入新的（避免污染主对话的 cache）
    })
    log(cacheReadTokens, cacheCreationTokens, hitRate)
else:
    // fallback：走普通 streaming API 调用，无 cache 共享
```

```
// 关键约束：cache key 的任何字段不一致都会 miss
// 以下字段必须与主对话完全一致：
//   system prompt、tools 列表、model ID、thinking config
// 如果 thinkingConfig 不同 → cache miss → 压缩成本显著上升
```

---

### Thinking Block 的特殊处理

**机制**：Extended thinking（extended thinking 模式）下，模型输出包含 `thinking` 块和 `text` 块。压缩时需要特别处理这两类块。

```ts
// 压缩调用时，强制禁用 thinking：
compactApiCall = {
    ...mainThreadParams,
    thinkingConfig: { type: "disabled" },  // 明确禁用
    // 原因 1：压缩本身不需要 thinking（任务是生成摘要，不是推理）
    // 原因 2：如果开启 thinking，thinking config 与主对话不同 → prompt cache miss
    // 原因 3：thinking 输出会增加摘要的 token 长度，得不偿失
}

// 历史消息中的 thinking 块如何处理：
// - 保留：thinking 块和 text 块必须成对（API 约束）
// - 压缩保留窗口计算时，thinking 块算在 token 估算中
// - 如果 startIndex 切在 thinking/text 对中间 → adjustIndex 往前移到完整对的起点
```

这与 API 不变量保护一致：不能只保留 `thinking` 块没有 `text`，也不能只有 `text` 没有 `thinking`（当 thinking 存在时）。

---

## API 结构不变量保护

压缩后的消息必须满足 Anthropic API 的结构约束：

| 约束 | 描述 | 违反后果 |
|------|------|---------|
| user/assistant 交替 | 消息必须以 user 开头，然后交替 | API 400 错误 |
| tool_use/tool_result 配对 | 每个 tool_use 必须有对应 tool_result | API 400 错误 |
| thinking block 完整性 | extended thinking 的 thinking/text 块必须成对 | 模型行为异常 |

```typescript
// Claude Code: sessionMemoryCompact.ts
function adjustIndexToPreserveAPIInvariants(
    messages: Message[],
    startIndex: number,
): number {
    // 如果 startIndex 处是 assistant message，往前找配对的 user message
    while (startIndex > 0 && messages[startIndex]?.type === 'assistant') {
        startIndex--;
    }
    // 如果 startIndex 是 tool_result，往前找到 tool_use 的起点
    // （tool_use 在 assistant message 中，tool_result 在下一个 user message 中）
    // 必须保证 tool_use/tool_result 对完整保留
    return startIndex;
}
```

```typescript
// OpenClaw: compact.ts
// sanitizeToolUseResultPairing(): 修复孤立的 tool_use 或 tool_result
// 孤立的 tool_use（没有对应 tool_result）→ 插入虚拟 tool_result
// 孤立的 tool_result（没有对应 tool_use）→ 删除
const repaired = sanitizeToolUseResultPairing(limited);
```

---

## 常量速查

```python
# OpenHarness
AUTOCOMPACT_BUFFER_TOKENS    = 13_000
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
COMPACT_TIMEOUT_SECONDS      = 25
MAX_COMPACT_STREAMING_RETRIES = 2
MAX_PTL_RETRIES              = 3
DEFAULT_KEEP_RECENT          = 5        # 微压缩保留条数
SESSION_MEMORY_KEEP_RECENT   = 12       # Session Memory 摘要保留条数
SESSION_MEMORY_MAX_LINES     = 48
SESSION_MEMORY_MAX_CHARS     = 4_000
CONTEXT_COLLAPSE_TEXT_CHAR_LIMIT = 2_400
CONTEXT_COLLAPSE_HEAD_CHARS  = 900
CONTEXT_COLLAPSE_TAIL_CHARS  = 500
```

```typescript
// Claude Code
const AUTOCOMPACT_BUFFER_TOKENS   = 13_000;
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000;
const MAX_PTL_RETRIES             = 3;
const MAX_CONSECUTIVE_FAILURES    = 3;
// Session Memory Compact 配置
const SM_COMPACT_MIN_TOKENS       = 10_000;  // 保留窗口最小 token
const SM_COMPACT_MAX_TOKENS       = 40_000;  // 保留窗口最大 token
const SM_COMPACT_MIN_TEXT_BLOCKS  = 5;       // 最少保留含文本的消息条数
```

```typescript
// OpenClaw
const DEFAULT_MAX_LIVE_TOOL_RESULT_CHARS = 40_000;  // 单条 tool result 上限
const MAX_TOOL_RESULT_CONTEXT_SHARE      = 0.3;     // tool results 总量最多占 30% context
const MIN_KEEP_CHARS                     = 2_000;   // 截断时最少保留
const EMBEDDED_COMPACTION_TIMEOUT_MS     = 900_000; // 15 分钟安全超时
```
