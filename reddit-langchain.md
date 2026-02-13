# r/LangChain — memsearch + LangChain/LangGraph

**Title:** Markdown-first semantic memory that plugs into LangChain and LangGraph — open source, backed by Milvus

---

Hey LangChain folks,

Sharing something we built that might be useful if you're building agents that need persistent memory. It's called [memsearch](https://github.com/zilliztech/memsearch) — a semantic search library where **your markdown files are the source of truth** and the vector store (Milvus) is just a derived index.

The idea: your agent writes observations to plain `.md` files, memsearch indexes them with content-hash dedup (unchanged chunks are never re-embedded), and later retrieves relevant context by cosine similarity. You can rebuild the entire index from scratch anytime since the markdown is always the canonical copy.

We just added LangChain and LangGraph integration examples and figured I'd share the patterns here.

**LangChain Retriever + RAG Chain:**

```python
import asyncio
from pydantic import ConfigDict
from memsearch import MemSearch
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from langchain_core.callbacks import CallbackManagerForRetrieverRun

class MemSearchRetriever(BaseRetriever):
    """LangChain retriever backed by memsearch."""
    mem: MemSearch
    top_k: int = 5
    model_config = ConfigDict(arbitrary_types_allowed=True)

    def _get_relevant_documents(
        self, query: str, *, run_manager: CallbackManagerForRetrieverRun
    ) -> list[Document]:
        results = asyncio.run(self.mem.search(query, top_k=self.top_k))
        return [
            Document(
                page_content=r["content"],
                metadata={"source": r["source"], "heading": r["heading"], "score": r["score"]},
            )
            for r in results
        ]
```

Then you can wire it into any LCEL chain:

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
answer = rag_chain.invoke("what caching solution are we using?")
```

**LangGraph ReAct Agent** — wrap memsearch as a tool and let the agent decide when to search:

```python
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def search_memory(query: str) -> str:
    """Search the team's knowledge base for relevant information."""
    results = asyncio.run(mem.search(query, top_k=3))
    if not results:
        return "No relevant memories found."
    return "\n\n".join(
        f"[{r['source']}] {r['heading']}: {r['content'][:300]}"
        for r in results
    )

agent = create_react_agent(llm, [search_memory])
result = agent.invoke({"messages": [("user", "what did the team work on last week?")]})
```

We also have LlamaIndex and CrewAI examples if anyone's using those.

The full integration docs with copy-paste code: [zilliztech.github.io/memsearch/integrations/](https://zilliztech.github.io/memsearch/integrations/)

Main things that differentiate this from just using a vanilla vector store:

- **Markdown is the source of truth** — no proprietary format, everything is git-diffable
- **Content-hash dedup** — SHA-256 hash as the primary key, so re-indexing unchanged files is essentially free
- **5 embedding providers** out of the box (OpenAI, Gemini, Voyage, Ollama, sentence-transformers) — including fully local options with zero API keys

Would be curious to hear if anyone's using a similar markdown-first approach for agent memory, or if most people just go straight to a vector DB. The "rebuildable index" property has saved us a few times when we switched embedding models and just re-ran `memsearch index`.
