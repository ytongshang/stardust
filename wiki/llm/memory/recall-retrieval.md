---
title: Recall & Retrieval
type: concept
created: 2026-04-15
updated: 2026-04-15
sources:
  - claude-code/src/memdir/findRelevantMemories.ts
  - claude-code/src/memdir/memoryScan.ts
  - claude-code/src/memdir/memdir.ts
  - openclaw/src/agents/memory-search.ts
  - openclaw/src/context-engine/types.ts
  - OpenHarness/src/openharness/memory/search.py
  - OpenHarness/src/openharness/memory/scan.py
  - OpenHarness/src/openharness/prompts/context.py
tags:
  - recall
  - retrieval
  - memory
  - RAG
  - agent
  - prompt-engineering
---

# Recall & Retrieval

## 核心问题

持久化的记忆只有在**正确时机**被**正确地**注入到 context 中才能发挥价值。过多注入浪费 token，注入不足让模型"失忆"。

召回系统需要回答三个问题：
1. **何时召回**：在哪个阶段触发检索
2. **召回什么**：选择哪些记忆条目
3. **如何注入**：以什么格式放入 context

---

## 一、Claude Code：LLM 语义选择召回

### 召回时机

每轮 query 时，在构建 system prompt 阶段触发记忆召回。

```
用户输入
  → 扫描所有记忆文件 header（只读 frontmatter，不读全文）
  → LLM（claude-sonnet）根据 query 和 header 选出最相关的 ≤5 条
  → 读取被选中文件的全文
  → 注入 system prompt
  → 模型推理
```

### 第一步：扫描 Memory Headers

```typescript
// memoryScan.ts — scanMemoryFiles()
// 只读取每个文件的 frontmatter 和前 30 行，构建 MemoryHeader 列表
interface MemoryHeader {
    filename: string;           // 相对路径（用于 LLM 选择器输出）
    filePath: string;           // 绝对路径（用于读取全文）
    mtimeMs: number;            // 修改时间（用于排序）
    description: string | null; // 从 frontmatter.description 读取
    type: MemoryType | undefined; // user | feedback | project | reference
}

async function scanMemoryFiles(
    memoryDir: string,
    signal: AbortSignal,
): Promise<MemoryHeader[]> {
    const files = await glob(`${memoryDir}/**/*.md`, { signal });
    const headers: MemoryHeader[] = [];

    for (const filePath of files) {
        const stat = await fs.stat(filePath);
        const content = await fs.readFile(filePath, 'utf8');
        const lines = content.split('\n').slice(0, 30);  // 只读前 30 行

        let description: string | null = null;
        let type: MemoryType | undefined;

        // 解析 YAML frontmatter
        if (lines[0]?.trim() === '---') {
            const endIdx = lines.indexOf('---', 1);
            if (endIdx > 0) {
                for (const line of lines.slice(1, endIdx)) {
                    const [key, ...rest] = line.split(':');
                    const value = rest.join(':').trim();
                    if (key.trim() === 'description') description = value;
                    if (key.trim() === 'type') type = value as MemoryType;
                }
            }
        }

        headers.push({
            filename: path.relative(memoryDir, filePath),
            filePath,
            mtimeMs: stat.mtimeMs,
            description,
            type,
        });
    }

    // 按修改时间降序（最新优先）
    return headers.sort((a, b) => b.mtimeMs - a.mtimeMs);
}
```

### 第二步：格式化 Memory Manifest

```typescript
// memoryScan.ts — formatMemoryManifest()
// 生成发送给 LLM 选择器的文件列表（每条一行）
function formatMemoryManifest(memories: MemoryHeader[]): string {
    return memories.map(m => {
        const timestamp = new Date(m.mtimeMs).toISOString();
        const typeStr = m.type ? `[${m.type}] ` : '';
        const desc = m.description ?? '(no description)';
        return `- ${typeStr}${m.filename} (${timestamp}): ${desc}`;
    }).join('\n');
}
```

生成的 manifest 格式：
```
- [feedback] feedback_testing.md (2026-04-14T14:22:00Z): 不要 mock 数据库，使用真实 DB 做集成测试
- [user] user_role.md (2026-04-13T10:00:00Z): 用户是 data scientist，关注 observability
- [project] api_architecture.md (2026-04-12T09:15:00Z): REST API 设计和认证流程
- [reference] grafana_dashboard.md (2026-04-10T16:30:00Z): oncall latency 监控面板地址
```

### 第三步：LLM 语义选择

这是 Claude Code 最核心的设计——用一次**轻量 LLM 调用**做语义匹配：

```typescript
// findRelevantMemories.ts

// System prompt：指导模型做精确、克制的选择
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful \
to Claude Code as it processes a user's query. You will be given the user's query and a \
list of available memory files with their filenames and descriptions.

Return a list of filenames for the memories that will clearly be useful to Claude Code \
as it processes the user's query (up to 5). Only include memories that you are certain \
will be helpful based on their name and description.
- If you are unsure if a memory will be useful in processing the user's query, then do \
not include it in your list. Be selective and discerning.
- If there are no memories in the list that would clearly be useful, feel free to return \
an empty list.
- If a list of recently-used tools is provided, do not select memories that are usage \
reference or API documentation for those tools (Claude Code is already exercising them). \
DO still select memories containing warnings, gotchas, or known issues about those tools \
— active use is exactly when those matter.`;

async function selectRelevantMemories(
    query: string,
    memories: MemoryHeader[],
    signal: AbortSignal,
    recentTools: string[],  // 当前轮次已使用的工具列表
): Promise<string[]> {
    const validFilenames = new Set(memories.map(m => m.filename));
    const manifest = formatMemoryManifest(memories);

    // 注入最近使用的工具信息
    const toolsSection = recentTools.length > 0
        ? `\n\nRecently used tools: ${recentTools.join(', ')}`
        : '';

    const result = await sideQuery({
        model: getDefaultSonnetModel(),          // 用 sonnet 节省成本
        system: SELECT_MEMORIES_SYSTEM_PROMPT,
        skipSystemPromptPrefix: true,            // 不注入 Claude Code 的系统提示
        messages: [{
            role: 'user',
            content: `Query: ${query}\n\nAvailable memories:\n${manifest}${toolsSection}`,
        }],
        max_tokens: 256,  // 回复很短（只需文件名列表）
        // 强制 JSON Schema 输出，防止模型输出无法解析的内容
        output_format: {
            type: 'json_schema',
            schema: {
                type: 'object',
                properties: {
                    selected_memories: {
                        type: 'array',
                        items: { type: 'string' },
                    },
                },
                required: ['selected_memories'],
                additionalProperties: false,
            },
        },
        signal,
        querySource: 'memdir_relevance',  // 标记来源，防止触发 session memory 提取
    });

    const textBlock = result.content.find(b => b.type === 'text');
    if (!textBlock || textBlock.type !== 'text') return [];

    const parsed: { selected_memories: string[] } = JSON.parse(textBlock.text);
    // 过滤幻觉：只接受 manifest 中存在的文件名
    return parsed.selected_memories.filter(f => validFilenames.has(f));
}
```

### 第四步：主入口函数

```typescript
// findRelevantMemories.ts — 对外暴露的主函数
async function findRelevantMemories(
    query: string,
    memoryDir: string,
    signal: AbortSignal,
    recentTools: readonly string[] = [],
    // alreadySurfaced：已经在本会话中注入过的记忆，防止重复注入
    alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]> {
    const memories = (await scanMemoryFiles(memoryDir, signal))
        .filter(m => !alreadySurfaced.has(m.filePath));

    if (memories.length === 0) return [];

    const selectedFilenames = await selectRelevantMemories(
        query, memories, signal, recentTools
    );
    const byFilename = new Map(memories.map(m => [m.filename, m]));

    // 可选：上报 telemetry（记忆召回形状分析）
    if (feature('MEMORY_SHAPE_TELEMETRY')) {
        logMemoryRecallShape(memories, selected);
    }

    return selectedFilenames
        .map(f => byFilename.get(f))
        .filter(Boolean)
        .map(m => ({ path: m.filePath, mtimeMs: m.mtimeMs }));
}
```

### 注入方式

被选中的记忆以 `<memory>` 标签注入 system prompt：

```
[原始 system prompt 内容]

[MEMORY.md 索引内容]

<memory path="memory/feedback_testing.md">
---
name: 不要在测试中 mock 数据库
description: 集成测试必须连真实 DB
type: feedback
---

不要 mock 数据库。

**Why:** 上季度 mock 测试全通过但 prod 迁移失败。
**How to apply:** 所有 DB 测试用 testcontainers。
</memory>
```

---

## 二、OpenHarness：关键词权重打分召回

### 召回时机

在每轮 query 构建 system prompt 时触发，分两层注入：

```python
# prompts/context.py — build_runtime_system_prompt()
def build_runtime_system_prompt(settings, *, cwd, latest_user_prompt=None, ...) -> str:
    sections = [base_system_prompt]
    # ... skills, delegation, CLAUDE.md 等固定内容

    if settings.memory.enabled:
        # 层 1：始终注入 MEMORY.md 索引（不需要 query，固定内容）
        memory_section = load_memory_prompt(
            cwd,
            max_entrypoint_lines=settings.memory.max_entrypoint_lines,
        )
        if memory_section:
            sections.append(memory_section)

        # 层 2：根据当前 query 动态召回相关记忆全文
        if latest_user_prompt:
            relevant = find_relevant_memories(
                latest_user_prompt,
                cwd,
                max_results=settings.memory.max_files,  # 默认 5
            )
            if relevant:
                lines = ["# Relevant Memories"]
                for header in relevant:
                    content = header.path.read_text(encoding="utf-8", errors="replace").strip()
                    lines.extend([
                        "",
                        f"## {header.path.name}",
                        "```md",
                        content[:8_000],  # 单条记忆最多 8000 chars（约 2000 tokens）
                        "```",
                    ])
                sections.append("\n".join(lines))

    return "\n\n".join(s for s in sections if s.strip())
```

注入后 system prompt 结构：
```
[Base System Prompt]

# Memory
- Persistent memory directory: /path/to/memory
- ...

## MEMORY.md
```md
- [反馈：不要 mock 数据库](feedback_testing.md) — ...
- [用户角色](user_role.md) — data scientist
```

# Relevant Memories

## feedback_testing.md
```md
[文件全文，最多 8000 chars]
```
```

### 记忆文件解析（scan.py）

```python
# memory/scan.py — _parse_memory_file()
def _parse_memory_file(path: Path, content: str) -> MemoryHeader:
    lines = content.splitlines()
    title = path.stem         # 默认用文件名
    description = ""
    memory_type = ""
    body_start = 0

    # 解析 YAML frontmatter（--- 到 --- 之间）
    if lines and lines[0].strip() == "---":
        for i, line in enumerate(lines[1:], 1):
            if line.strip() == "---":
                for fm_line in lines[1:i]:
                    key, _, value = fm_line.partition(":")
                    key, value = key.strip(), value.strip().strip("'\"")
                    if not value: continue
                    if key == "name": title = value
                    elif key == "description": description = value
                    elif key == "type": memory_type = value
                body_start = i + 1
                break

    # Fallback：用正文第一个非空、非标题行作为 description
    desc_line_idx = None
    if not description:
        for idx, line in enumerate(lines[body_start:body_start + 10], body_start):
            stripped = line.strip()
            if stripped and stripped != "---" and not stripped.startswith("#"):
                description = stripped[:200]
                desc_line_idx = idx
                break

    # 正文预览（用于搜索打分）
    body_lines = [
        line.strip()
        for idx, line in enumerate(lines[body_start:], body_start)
        if line.strip()
        and not line.strip().startswith("#")
        and idx != desc_line_idx  # 排除已用作 description 的行
    ]
    body_preview = " ".join(body_lines)[:300]

    return MemoryHeader(
        path=path,
        title=title,
        description=description,
        modified_at=path.stat().st_mtime,
        memory_type=memory_type,
        body_preview=body_preview,
    )
```

### 相关性打分算法

```python
# memory/search.py
def find_relevant_memories(
    query: str,
    cwd: str | Path,
    *,
    max_results: int = 5,
) -> list[MemoryHeader]:
    tokens = _tokenize(query)
    if not tokens:
        return []

    scored = []
    for header in scan_memory_files(cwd, max_files=100):
        # 搜索字段：标题 + description（元数据）
        meta = f"{header.title} {header.description}".lower()
        # 搜索字段：正文预览（前 300 chars）
        body = header.body_preview.lower()

        # 元数据命中权重 2x，正文命中权重 1x
        # 这激励写好 description：description 好 ≈ 召回准确率高
        meta_hits = sum(1 for t in tokens if t in meta)
        body_hits = sum(1 for t in tokens if t in body)
        score = meta_hits * 2.0 + body_hits

        if score > 0:
            scored.append((score, header))

    # 按分数降序，同分按修改时间降序（最新优先）
    scored.sort(key=lambda item: (-item[0], -item[1].modified_at))
    return [header for _, header in scored[:max_results]]


def _tokenize(text: str) -> set[str]:
    """支持 ASCII 单词和 CJK 字符的 tokenization"""
    # ASCII：3 个字符以上的单词（过滤 "a", "to", "is" 等停用词）
    ascii_tokens = {t for t in re.findall(r"[A-Za-z0-9_]+", text.lower()) if len(t) >= 3}
    # CJK：每个字符独立作为 token（中文词边界不确定，字粒度最安全）
    han_chars = set(re.findall(r"[\u4e00-\u9fff\u3400-\u4dbf]", text))
    return ascii_tokens | han_chars
```

---

## 三、OpenClaw：向量 + 全文混合（Hybrid RAG）

### 召回时机

通过 Context Engine 的 `assemble()` 方法触发，是可插拔的接口：

```typescript
// context-engine/types.ts

interface ContextEngine {
    // 每轮 query 前调用，返回组装好的 context
    assemble(params: {
        sessionId: string;
        sessionKey?: string;
        messages: AgentMessage[];
        tokenBudget?: number;
        availableTools?: Set<string>;     // 当前可用工具（影响记忆选择）
        citationsMode?: MemoryCitationsMode;
        model?: string;                   // 模型 ID（不同模型可能有不同格式偏好）
        prompt?: string;                  // 当前用户输入（用于语义检索）
    }): Promise<AssembleResult>;
}

// 返回值
interface AssembleResult {
    messages: AgentMessage[];        // 组装后的消息列表（含历史 + 记忆）
    estimatedTokens: number;         // 预计 token 数
    systemPromptAddition?: string;   // 注入 system prompt 的额外内容（含检索到的记忆）
}
```

### 搜索实现（memory-search.ts）

OpenClaw 的搜索配置决定了如何构建搜索 query 并返回结果：

```typescript
// memory-search.ts — 搜索配置
const searchConfig = {
    // 混合搜索：向量（语义相似度）+ BM25（全文关键词）
    type: "hybrid",
    vectorWeight: 0.7,     // 70% 向量权重：捕捉语义相关性
    textWeight: 0.3,       // 30% BM25 权重：捕捉精确关键词匹配
    candidateMultiplier: 4, // 先取 maxResults * 4 = 24 个候选，再 rerank

    // MMR (Maximal Marginal Relevance)：结果多样性去重
    // score(d) = λ·sim(query, d) - (1-λ)·max_{d'∈Selected} sim(d, d')
    mmr: {
        enabled: false,    // 默认关闭，可配置开启
        lambda: 0.7,       // 相关性 vs 多样性的权衡（0=最多样，1=最相关）
    },

    // 时序衰减：越新的记忆权重越高
    temporalDecay: {
        enabled: false,    // 默认关闭
        halfLifeDays: 30,  // 30 天后相关性权重降为 50%
    },

    maxResults: 6,
    minScore: 0.35,        // 低于此阈值的结果直接丢弃
};

// 文本分块策略（长文本存入 SQLite 前切块）
const chunkingConfig = {
    defaultChunkTokens: 400,  // 每块约 400 token
    overlapTokens: 80,        // 相邻块重叠 80 token，避免语义在块边界断裂
    ftsTokenizer: "unicode61", // SQLite FTS5 tokenizer（支持 Unicode）
};
```

**MMR 算法详解：**

```
普通 Top-K 召回的问题：
  - "如何使用 bash" (sim=0.9)
  - "bash 命令示例" (sim=0.88)    ← 与第 1 条几乎重复
  - "bash 脚本写法" (sim=0.85)    ← 与第 1 条几乎重复

MMR 召回（λ=0.7）：
  Step 1: 选 sim 最高的 "如何使用 bash" (0.9)
  Step 2: 对剩余文档计算 MMR 分数：
    "bash 命令示例": 0.7*0.88 - 0.3*max(sim("bash 命令示例","如何使用 bash"))
                  = 0.616 - 0.3*0.92 = 0.340  ← 因为与已选高度相似，分数大幅下降
    "已知 bash 问题": 0.7*0.75 - 0.3*0.40 = 0.525 - 0.12 = 0.405  ← 内容不重复，分数较高
  Step 2 结果：选 "已知 bash 问题"

最终：选出相关且多样的两条，而不是两条几乎重复的内容
```

### 搜索来源配置

```typescript
// 可搜索两种来源
type MemorySource = "memory" | "sessions";

// "memory" 源：搜索 memory 目录下的 .md 文件
// "sessions" 源：搜索历史 session transcripts（强大功能！）
//   → 可以找到"上次讨论这个 API 是在哪个会话里"
//   → 可以跨会话检索历史代码片段

const sources: MemorySource[] = ["memory", "sessions"];
```

---

## 注入格式对比

### Claude Code 注入格式

每条召回的记忆文件全文以 `<memory>` 标签注入 system prompt：

```
<memory path="memory/feedback_testing.md">
[文件全文]
</memory>
```

- **MEMORY.md 索引**始终注入（固定内容，走 prompt cache）
- **记忆全文**按需动态注入（随 query 变化）
- 限制：每条记忆无显式字数限制（由 memdir 文件大小隐性控制）

### OpenHarness 注入格式

```
# Memory
- Persistent memory directory: /path
- ...

## MEMORY.md
```md
[MEMORY.md 索引内容，最多 200 行]
```

# Relevant Memories

## feedback_testing.md
```md
[文件全文，最多 8000 chars]
```

## user_role.md
```md
[文件全文，最多 8000 chars]
```
```

- **MEMORY.md 索引**始终注入（固定内容）
- **记忆全文**动态注入，每条 **≤8000 chars**（显式 cap）

### OpenClaw 注入格式

通过 `AssembleResult.systemPromptAddition` 返回，格式由 Context Engine 实现决定：

```typescript
// 典型格式（内置实现）
interface AssembleResult {
    messages: AgentMessage[];
    estimatedTokens: number;
    systemPromptAddition?: string;  // 含记忆引用的 Markdown 片段
}

// 注入内容示例
systemPromptAddition = `
## Memory Context
Based on your query, the following stored context may be relevant:

[feedback_testing.md] — 不要 mock 数据库：集成测试必须连真实 DB...

[timestamp: 2026-04-14, similarity: 0.82]
`;
```

---

## 防止过度召回

### alreadySurfaced 去重（Claude Code）

跨轮次追踪已注入的记忆，不在同一会话中重复注入：

```typescript
// 在 context 构建器中维护
const alreadySurfaced = new Set<string>();

// 每轮 query 时
const newMemories = await findRelevantMemories(
    query, memoryDir, signal, recentTools,
    alreadySurfaced,  // 过滤掉已经注入过的
);
// 更新集合
newMemories.forEach(m => alreadySurfaced.add(m.path));
```

### 工具使用去重（Claude Code）

系统 prompt 中已明确：如果最近正在**使用**某工具，不召回该工具的 API 文档，但**仍召回**该工具的已知问题：

```
- If recently-used tools list includes "bash":
  ✗ 不召回: "bash_usage_guide.md"     （API 文档 — 模型已在用了）
  ✓ 仍召回: "bash_path_spaces_bug.md"  （已知 bug — 正在用时最需要）
```

### Token Budget 控制（OpenHarness）

```python
# 每条记忆内容 8000 chars ≈ 2000 tokens
content = header.path.read_text().strip()
lines.extend(["```md", content[:8_000], "```"])

# MEMORY.md 索引最多 200 行
content_lines = entrypoint.read_text().splitlines()[:max_entrypoint_lines]
```

### MMR 多样性控制（OpenClaw）

用算法保证召回结果不重复，比手动设置数量上限更精细。

---

## 三方案对比

| 特性 | Claude Code | OpenHarness | OpenClaw |
|------|------------|------------|---------|
| **召回算法** | LLM 语义选择 | 关键词权重打分 | 向量+全文混合 |
| **需要 LLM** | 是（小模型 sonnet） | 否 | 否（SQLite 查询）|
| **延迟** | 中（额外 API 调用） | 低（纯 CPU） | 低 |
| **语义理解** | 强 | 弱 | 中 |
| **精确关键词** | 中（LLM 推理） | 强 | 强（BM25） |
| **多样性控制** | 数量限制（≤5） | 数量限制 | MMR 算法 |
| **时序偏好** | mtime 排序 | mtime 排序 | 时序衰减权重 |
| **跨 session** | 否 | 否 | 是（sessions 源）|
| **防重复注入** | alreadySurfaced 集合 | 无 | MMR 天然去重 |
| **索引格式** | YAML frontmatter | YAML frontmatter | SQLite chunks |
| **单条大小限制** | 无显式限制 | 8000 chars | 按 token 分块 |

### 什么时候选哪种

| 场景 | 推荐 | 理由 |
|------|------|------|
| 记忆文件少（<50），但需要理解语义 | Claude Code | LLM 选择器适合小集合+语义理解 |
| 记忆文件多（>100），低延迟优先 | OpenClaw | 向量索引在大集合下效率更高 |
| 快速实现，无向量数据库 | OpenHarness | 只需 Python 标准库，无依赖 |
| 需要跨历史会话检索 | OpenClaw | sessions 源支持跨 session 搜索 |
| 记忆条目高度同质（如大量代码片段） | OpenClaw | MMR 保证多样性 |
