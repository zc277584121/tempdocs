# r/ClaudeCode — Progressive Disclosure for Agent Memory

**Title:** We built a memory plugin for Claude Code that uses progressive disclosure — curious what you all think

---

Hey everyone,

We've been working on an open-source plugin that gives Claude Code persistent memory across sessions. The core idea isn't novel — save past conversations to markdown, embed them, and retrieve relevant context later. But the part I'm most excited about is how we handle *retrieval depth*, and I'd love to hear if anyone's thinking about this differently.

**The problem we kept running into:** naive memory injection either gives the agent too little context (just a snippet) and it hallucinates the rest, or too much context (dump everything) and it blows up the context window. We landed on a three-layer approach that we're calling "progressive disclosure":

**Layer 1 — Auto-injected.** On every prompt, a hook automatically searches past memories and injects the top-3 results as short previews (~200 chars each). Claude sees them before it even starts thinking. No tool call needed, no "should I look up my memory?" decision. This alone handles maybe 80% of cases.

**Layer 2 — Expand on demand.** When a preview isn't enough, Claude can run `memsearch expand <chunk_hash>` to pull the full markdown section surrounding that memory. This is where you get the full bullet list, the neighboring context, the decision rationale — stuff that was too long for L1 but is now specifically requested.

**Layer 3 — Transcript drill-down.** Each memory chunk links back to the original Claude Code session transcript (the JSONL files). When Claude needs the *exact* code snippet or error message from a past conversation, it can drill all the way down to the raw conversation turns. We've never actually seen it go to L3 unprompted, but when you ask "what was that exact error from yesterday?" it knows how to get there.

The key insight for us was that L1 being *automatic* matters way more than we expected. In earlier versions we had Claude call a search tool, but it would just... forget to look things up. Or it would search for the wrong thing. Having the hook inject context *before* Claude processes the message means memories are never missed.

Everything is markdown-first — the vector store (Milvus) is just a derived index that can be rebuilt anytime. Your memory files are plain `.md` that you can read, edit, grep, and git-diff. No vendor lock-in.

The plugin is fully open source: [github.com/zilliztech/memsearch](https://github.com/zilliztech/memsearch)

Docs for the plugin and the progressive disclosure model: [zilliztech.github.io/memsearch/claude-plugin/](https://zilliztech.github.io/memsearch/claude-plugin/)

A few open questions I'm genuinely curious about:

- Has anyone tried other approaches to memory retrieval depth? We considered a single "adaptive" layer that returns variable-length context, but it was hard to get right without over-fetching.
- For those using CLAUDE.md files as memory — do you hit the size limit often? That's the main thing progressive disclosure is solving for us: keeping context usage minimal while still having deep history available.
- Anyone else working on hook-based context injection? The `UserPromptSubmit` hook feels underutilized for this kind of thing.

Would love to hear what's working (or not working) for others. Happy to answer any questions about the implementation.
