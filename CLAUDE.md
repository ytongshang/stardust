# Stardust Wiki

> Schema document — read at session start with `wiki/index.md` and the relevant domain index.
> Update after every major compile, ingest batch, or structural change.

## Scope

What this wiki covers:
- AI / LLM 技术：模型原理、Prompt 工程、Caching、推理优化
- Agent 系统：架构设计、工具设计、Skills、文件系统、多 Agent 协作
- 软件工程：系统设计、基础设施、开发工具
- 其他个人研究兴趣（按主题积累后建新 domain）

What this wiki deliberately excludes:
- 日记、流水账、临时笔记
- 未经精读的来源（先放 `raw/<domain>/`，ingest 后才进 `wiki/`）
- 与研究主题无关的工具使用手册

## Wiki structure

| Domain | Path | Scope |
|--------|------|-------|
| llm | `wiki/llm/` | AI / LLM / Agent 系统 |
| life | `wiki/life/` | 日常生活、个人工作流、自动化 SOP |

To add a new domain: create `wiki/<domain>/` with the same structure, add an entry to `wiki/index.md`.

## Naming conventions

- **Concept pages** (`wiki/<domain>/concepts/`): Title Case noun phrases，中文主题用中文标题。
- **Folder-split concepts** (`wiki/<domain>/concepts/<topic>/`): when a topic has multiple independent aspects. Contains `index.md` + one file per aspect.
- **Entity pages** (`wiki/<domain>/entities/`): 人名/工具名/论文名用原名（英文）。
- **Summary pages** (`wiki/<domain>/summaries/`): kebab-case source slug。

All pages require YAML frontmatter: `title`, `type`, `created`, `updated`, `sources`, `tags`.

### Wikilinks
- Always `[[Page Title]]` — exact title, case-sensitive.
- Folder-split: link the index `[[<domain>/concepts/Foo/index|Foo]]`.
- Link first mention of every entity/concept. Max twice per article.

### Diagrams and formulas
- Diagrams: **mermaid** only. No ASCII art.
- Formulas: **KaTeX** — inline `$...$` or block `$$...$$`.

### Raw file policy
- Small text sources → `raw/<domain>/<type>/` (articles, papers, notes).
- Large binaries → pointer file at `raw/<domain>/refs/<slug>.md` with `kind: ref` + `external_path`. Do not copy.
- **article2md skill**：使用 `/article2md` 抓取文章时，产物直接写入 `raw/<domain>/<type>/` 目录（如 `raw/llm/articles/`），不再调用额外工具移动文件。文件名使用 kebab-case slug。

### Language
- 中英文双语：概念解释优先中文，专业术语保留英文原名。
- Summary 语言跟随原文，中文说明部分用中文。

## Current articles

*Last updated: 2026-04-12*

See `wiki/llm/index.md` for full listing. Summary:

### Domain: llm (`wiki/llm/`)
- Concepts: agent-file-system, agent-tool-design, agent-skills, claude-prompting, prompt-caching/ (folder-split, 4 pages), agents-harness/ (folder-split, 4 pages: what-is-harness, core-components, guides-vs-sensors, harness-engineering-lessons)
- Concepts: multi-agent-workflow/ (folder-split, 5 pages: index, routing, roles, checkpoints, artifacts) — S0/S1/S2 personal automation workflow for minimal-agent collaboration, verification, and knowledge migration.
- Life workflows: request-to-automation — short SOP for turning a personal/work request into implementation, verification, and knowledge migration.
- Memory: memory/ (folder-split, 5 pages) — context-compression, memory-persistence, recall-retrieval, implementation-comparison
- Entities: Thariq Shihipar
- Summaries: 12 sources ingested (8 prior + 4 agents-harness: langchain-anatomy-agent-harness, martin-fowler-harness-engineering, philipp-schmid-agent-harness-2026, parallel-what-is-agent-harness)

## Open research questions

- Agent 系统在大规模工具集下如何保持 cache prefix 稳定？
- llm-wiki 模式在团队场景下的 audit 工作流如何设计？

## Research gaps

Sources to ingest:
- [ ] raw/llm/articles/trq212-agent-file-system.md — 待精读：Your Agent should use a File System
- [ ] raw/llm/articles/trq212-agents-need-bash.md — 待精读：Why even non-coding agents need bash

## Audit backlog

*(run `python3 scripts/audit_review.py <wiki-root> --open` to refresh)*

## Skill: article2md

抓取网页文章并转换为 Markdown。脚本通过 `uv run` 执行，无需手动装依赖。

| Source | Script | Domain |
|---|---|---|
| 微信公众号 | `fetch_weixin.py` | `mp.weixin.qq.com` |

用法示例：
```bash
uv run .claude/skills/article2md/scripts/fetch_weixin.py <url> -o raw/<domain>/articles/<slug>.md
```

**工作流**：根据 URL 域名选择脚本 → 执行抓取 → 通过 `-o` 参数将产物直接写入 `raw/<domain>/<type>/<slug>.md` → 更新 CLAUDE.md Research gaps 待 ingest 列表。

## Notes for the LLM

- Language: bilingual（中英文双语）
- Tone: neutral, technical
- Depth: deep technical
- Handling contradictions: state both, cite each, add to Open Research Questions.
