---
title: "Giving Claude Code Permanent Memory — and Why CLAUDE.md Isn't Enough"
published: false
tags: ai, claudecode, milvus, agents
---

I've been using Claude Code as my primary coding assistant for a few months now. It's genuinely great — until you start a new session and it has no idea what you talked about yesterday. You find yourself re-explaining architecture decisions, re-describing bugs you already fixed, re-stating team conventions.

Claude Code *does* have a memory system: `CLAUDE.md` files and the `/memory` command. But as your project history grows, this single-file approach starts to crack. So we built [memsearch](https://github.com/zilliztech/memsearch) — an open-source plugin that gives Claude Code **real persistent memory** with semantic search, automatic session capture, and a three-layer progressive disclosure model.

In this post I'll walk through why we built it, how it works, and how it compares to Claude Code's native memory and another popular solution called [claude-mem](https://github.com/thedotmack/claude-mem).

## The Problem with CLAUDE.md

Claude Code's built-in memory works like this: there's a `CLAUDE.md` file at your project root. You write stuff in it, and Claude sees it at the start of every session. Simple and effective — for a while.

Here's what happens over time:

| Week 1 | Week 8 |
|--------|--------|
| CLAUDE.md is 20 lines | CLAUDE.md is 400+ lines |
| Everything is relevant | 90% is noise for any given task |
| Loads instantly | Eats a chunk of your context window |
| Easy to maintain | Nobody wants to prune it |

The fundamental issue is that `CLAUDE.md` has **no search**. It loads everything or nothing. Claude can't selectively recall "what was that Redis config decision from two weeks ago?" — it either happens to be in the file, or it's gone.

And if you're using the `/memory` command to auto-append, the file grows monotonically. There's no dedup, no summarization, no relevance filtering.

## How memsearch Works

memsearch takes a different approach: **your memory lives in daily markdown files**, and a vector index makes them searchable.

```
your-project/
├── .memsearch/
│   └── memory/
│       ├── 2026-02-07.md     ← daily session log
│       ├── 2026-02-08.md
│       └── 2026-02-09.md
└── ... (your project files)
```

Each daily file contains bullet-point summaries of what happened in each Claude Code session. These are generated **automatically** — when you end a session, a hook summarizes the conversation using Haiku and appends it to today's file.

The full data flow:

```mermaid
sequenceDiagram
    participant U as You
    participant C as Claude Code
    participant H as memsearch hooks
    participant M as Milvus (vector DB)

    U->>C: Start session
    H->>H: Start background watcher
    H->>M: Index existing memory files

    U->>C: "How should we configure Redis?"
    H->>M: Semantic search (automatic)
    M-->>H: Top-3 relevant memories
    H->>C: Inject memories into prompt
    C->>U: Answer with context from past sessions

    U->>C: End session
    H->>H: Summarize transcript (Haiku)
    H->>H: Append to today's .md file
    H->>M: Index new content
```

The key pieces:

1. **Automatic capture.** A `Stop` hook fires when you close a session, parses the conversation transcript, calls Haiku for a summary, and writes it to `memory/YYYY-MM-DD.md`.

2. **Automatic recall.** A `UserPromptSubmit` hook fires on every message you send. It runs semantic search against all past memories and injects the top-3 results *before* Claude sees your message. No tool call, no decision-making — just relevant context, every time.

3. **Markdown as source of truth.** The vector store (Milvus) is a derived index. You can read, edit, `grep`, and `git diff` your memory files. Blow away the index? Just run `memsearch index` and it rebuilds in seconds.

## Progressive Disclosure: Three Layers of Context

This is the part I'm most proud of. Dumping all relevant memories into the prompt would waste the context window. Showing only tiny snippets might not be enough. So we use a three-layer approach:

```mermaid
graph LR
    L1["L1: Auto-injected previews<br/>~200 chars each"] -->|"Need more?"| L2["L2: Full section<br/>memsearch expand"]
    L2 -->|"Need the original?"| L3["L3: Raw transcript<br/>memsearch transcript"]
```

**Layer 1** is automatic. On every prompt, Claude gets something like this:

```
## Relevant Memories
- [memory/2026-02-08.md:14:30]  Implemented two-tier caching: Redis L1
  with 5min TTL, in-process LRU L2 with 1000 entry cap. Cache invalidation
  via Redis pub/sub...
  `chunk_hash: 47b5475122b992b6`
```

Short previews. Enough for most follow-ups. ~200 characters per chunk, 3 chunks injected. Minimal context cost.

**Layer 2** — when the preview isn't enough, Claude runs:

```bash
$ memsearch expand 47b5475122b992b6
```

This returns the full markdown section, including neighboring headings and the session anchor metadata. Claude now sees the complete bullet list and surrounding context from that day.

**Layer 3** — when Claude needs the *exact* code or error message from a past session, it can drill into the raw transcript:

```bash
$ memsearch transcript /path/to/session.jsonl --turn 6d6210b7 --context 2
```

This pulls the original conversation turns — user messages, assistant responses, tool calls, everything. We rarely see Claude go to L3 unprompted, but when you ask "what was the exact error message from yesterday's deploy?", it knows the path.

The beauty is that **L1 handles ~80% of cases** with near-zero context cost. L2 and L3 are there for the remaining 20% and only get invoked when needed.

## Comparison: memsearch vs. CLAUDE.md vs. claude-mem

There are three approaches to Claude Code memory today. Here's an honest comparison:

| | CLAUDE.md | claude-mem | memsearch |
|---|-----------|------------|-----------|
| **How memory is recalled** | Entire file loaded at session start | Claude calls MCP search tool | Hook auto-injects top-k results |
| **Recall is...** | Always-on but unfiltered | On-demand (Claude must decide to search) | Always-on and filtered by relevance |
| **Storage format** | Single monolithic file | SQLite + Chroma (binary) | Daily `.md` files (human-readable) |
| **Search capability** | None | Dense vector search | Hybrid search (dense + BM25) |
| **History depth** | Limited to one file | Unlimited | Unlimited |
| **Automatic capture** | Manual (`/memory` command) | Automatic observations | Automatic session summaries |
| **Context window cost** | Grows linearly with file size | MCP tool definitions always loaded | Minimal — only top-k snippets |
| **Progressive disclosure** | None | 3-layer, all agent-driven | 3-layer, L1 is automatic |
| **Can rebuild index?** | N/A | No — Chroma is the data store | Yes — markdown is the source of truth |
| **Git-friendly?** | Yes (single file) | No (binary data) | Yes (one `.md` per day) |

The biggest differentiator between memsearch and claude-mem is **push vs. pull**.

claude-mem gives Claude MCP tools to search, explore timelines, and fetch observations. All three layers require Claude to *decide* to use them. In practice, Claude often doesn't call the search tool unless the conversation explicitly references past work. Relevant context gets silently missed.

memsearch **pushes** memories to Claude before every message. Claude doesn't need to decide whether to look things up — it just *has* the relevant context. This is a subtle but important difference. After using both approaches, we found that automatic injection catches context that agent-driven recall misses maybe 30-40% of the time.

## Quick Start

Install memsearch and set up the Claude Code plugin:

```bash
pip install memsearch
export OPENAI_API_KEY="sk-..."

# In Claude Code:
/plugin install memsearch
# Restart Claude Code for the plugin to take effect
```

That's it. From now on:

- Every session is automatically summarized and saved
- Every prompt gets relevant memories injected
- Your memory lives in `.memsearch/memory/*.md` — browse it anytime

If you want to try memsearch outside Claude Code, the Python API is three lines:

```python
import asyncio
from memsearch import MemSearch

async def main():
    mem = MemSearch(paths=["./memory/"])
    await mem.index()
    results = await mem.search("Redis config", top_k=3)
    for r in results:
        print(f"[{r['score']:.2f}] {r['source']} — {r['content'][:80]}")

asyncio.run(main())
```

## What's Next

We're actively working on this and would love feedback. Some things on the roadmap:

- **Memory compaction** — LLM-powered summarization to compress old memories (already in progress)
- **Multi-agent shared memory** — point multiple Claude Code instances at the same Milvus server
- **Smarter L1 injection** — adjusting the number of injected results based on query relevance scores

The whole project is open source: **[github.com/zilliztech/memsearch](https://github.com/zilliztech/memsearch)**

Full docs including the progressive disclosure deep-dive: **[zilliztech.github.io/memsearch](https://zilliztech.github.io/memsearch/)**

If you're using Claude Code and have been frustrated by the memory gap between sessions, give it a try. And if you've built your own memory solution, I'd genuinely love to hear about your approach — this is still a very unsettled design space.
