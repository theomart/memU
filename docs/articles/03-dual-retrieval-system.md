🎯 RAG is fast but dumb. LLM ranking is smart but slow. Why not both?

MemU's dual retrieval system lets you pick your tradeoff. Here's how it works.

---

👞 Non technical TLDR

- Imagine searching your email: you can do keyword search (fast, misses context) or have an assistant read everything and find what's relevant (accurate, takes forever)
- MemU gives you both options: `method="rag"` for speed, `method="llm"` for accuracy
- Spoiler: there's a smart "sufficiency check" that stops searching when you have enough info

---

🔬 Technical TLDR

- **RAG Method** (`method="rag"`):
  - Embed query → cosine similarity search → return top-k
  - ~50ms for 10k vectors (NumPy optimized with `argpartition`)
  - Good for: high-volume, latency-sensitive, keyword-ish queries

- **LLM Method** (`method="llm"`):
  - Present candidates as text → LLM ranks by relevance → return ranked results
  - ~2-5s depending on candidate count
  - Good for: complex queries, pronoun resolution, semantic understanding

- **Both use the same 3-tier hierarchy**:
  1. Categories (high-level summaries)
  2. Memory Items (specific facts)
  3. Resources (original documents)

---

🥊 The Sufficiency Check (this is the real magic)

After each tier, the system asks: "Do we have enough info to answer this query?"

```
Query: "What's the user's favorite food?"
→ Search Categories → Found "preferences" category with "loves Italian food"
→ Sufficiency Check: "YES, we have enough"
→ STOP (don't search items, don't search resources)
```

Result: faster responses, less noise, lower cost.

---

⏳ Latency estimates

| Operation | RAG | LLM |
|-----------|-----|-----|
| Category search | ~10ms | ~1s |
| Item search | ~30ms | ~2s |
| Resource search | ~50ms | ~3s |
| **Total (worst case)** | **~100ms** | **~6s** |

But with sufficiency checking, most queries stop at tier 1 or 2.

---

🛠 Pre-Retrieval Decision

Before ANY search, the system decides: "Does this query even NEED memory retrieval?"

- "Hello!" → NO_RETRIEVE (greeting)
- "What did I tell you about my project?" → RETRIEVE (references past)
- "What's 2+2?" → NO_RETRIEVE (general knowledge)

This alone saves ~30% of retrieval calls in production.

---

🍓 Key insight

Query rewriting happens at each tier. "What did she say about that?" becomes "What did Sarah say about the project deadline?" using conversation context.

= way better retrieval accuracy without user effort.

---

RAG vs LLM retrieval - which do you use in production? Share your thoughts :)

#ai #llm #rag #retrieval
