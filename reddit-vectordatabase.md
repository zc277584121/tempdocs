# r/VectorDatabase — Milvus as a derived index for agent memory

**Title:** Using Milvus as a "rebuildable index" for AI agent memory — markdown stays the source of truth

---

Wanted to share a pattern we've been using that flips the usual vector DB workflow on its head. Most RAG setups treat the vector store as the canonical data source — you ingest documents, and if the index gets corrupted or you want to switch models, it's a painful re-ingestion process. We went the other way: **markdown files are the source of truth, and Milvus is a derived index** — like a database index that you can drop and rebuild anytime.

The project is [memsearch](https://github.com/zilliztech/memsearch), a semantic memory library for AI agents. The workflow is:

1. Agent writes observations to plain `.md` files (one per day, OpenClaw-style)
2. memsearch scans the files, chunks by heading structure, and embeds each chunk
3. SHA-256 content hash is the Milvus primary key — so unchanged chunks are automatically deduped and never re-embedded
4. Search uses Milvus hybrid search (dense embedding + BM25 sparse) with RRF fusion

The "rebuildable index" thing has been genuinely useful in practice. We've switched embedding providers twice during development (OpenAI → local sentence-transformers → back to OpenAI) and each time it was just `memsearch index ./memory/` — full re-embed from the markdown, zero data loss, no migration scripts.

Some Milvus-specific things we learned:

- **Content hash as VARCHAR PK** works great for dedup. On re-index, if the hash already exists, we skip the embedding API call entirely. This makes incremental indexing very cheap.
- **Milvus Lite** (the embedded mode with a `.db` file) is surprisingly good for single-user/single-agent use cases. We default to it for zero-config local setups, and then people can point at a Milvus Server or Zilliz Cloud for shared/team deployments just by changing the URI.
- **`get_collection_stats()` lies after upsert** on Milvus Server — counts aren't updated until segment flush/compaction. Search works fine though. This confused us for a while during testing.
- **Hybrid search with `RRFRanker`** — we use both dense vectors and BM25 sparse vectors. The sparse side helps a lot with exact keyword matching (e.g., searching for a specific function name or config value that embedding models don't capture well).

The library supports 5 embedding providers (OpenAI, Gemini, Voyage, Ollama, sentence-transformers) and all 3 Milvus deployment modes (Lite/Server/Zilliz Cloud) — same codebase, just a config change.

GitHub: [github.com/zilliztech/memsearch](https://github.com/zilliztech/memsearch)
Docs: [zilliztech.github.io/memsearch](https://zilliztech.github.io/memsearch/)

Curious if anyone else is using vector DBs as derived/secondary indexes rather than as the primary data store. The "you can always rebuild it" guarantee has changed how we think about data durability — we don't worry about Milvus backups at all because the `.md` files are what matters (and those are in git).
