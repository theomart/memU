🦎 AI memory storage cheat sheet for practical deployments.

MemU supports 3 backends: InMemory, SQLite, PostgreSQL. Here's when to use each.

---

👞 Non technical TLDR

- Development? Use InMemory (zero setup, data disappears on restart)
- Single user app? Use SQLite (one file, portable, good enough)
- Production? Use PostgreSQL + pgvector (scales, native vector search = go brrrr)

---

🔬 Technical TLDR

- **All backends implement the same `Database` protocol** → swap with one config change
- Protocol-based design = duck typing, no inheritance, easy mocking for tests

---

📊 Backend comparison:

| | InMemory | SQLite | PostgreSQL |
|---|---|---|---|
| Persistence | ❌ | ✅ File | ✅ Server |
| Setup | Zero | DSN string | Full server |
| Vector search | NumPy cosine | Brute-force | pgvector HNSW |
| 10k vector search | ~50ms | ~100ms | ~5ms |
| Concurrency | Single process | Single writer | Full ACID |

---

⚡ pgvector is the real upgrade

Without pgvector:
- Store embeddings as JSON arrays
- Brute-force cosine similarity in Python
- O(n) for every search

With pgvector:
- Native `vector(1536)` column type
- HNSW approximate nearest neighbor index
- O(log n) search, SIMD acceleration

To put that in perspective: 10k vectors goes from ~100ms to ~5ms. At 1M vectors, it's the difference between "works" and "unusable".

---

🛠 Configuration examples

**Development:**
```python
store = InMemoryStore()
```

**Lightweight deployment:**
```python
store = SQLiteStore(
    dsn="sqlite:///./data/memu.sqlite"
)
```

**Production:**
```python
store = PostgresStore(
    dsn="postgresql://user:pass@db:5432/memu",
    vector_provider="pgvector",  # ← the magic flag
    ddl_mode="create"
)
```

---

👨‍🍳 Recipe for multi-tenant setup

MemU supports **scope models** for tenant isolation:

```python
class MyScope(BaseModel):
    user_id: str
    org_id: str | None = None

store = PostgresStore(
    dsn="...",
    scope_model=MyScope  # ← all tables get user_id, org_id columns
)

# Query with automatic filtering
items = store.memory_item_repo.list_items(
    where={"user_id": "user_123"}
)
```

No manual tenant filtering. No data leaks. Clean.

---

🍓 Key insight

The `Database` protocol means you can start with InMemory for prototyping, graduate to SQLite for MVP, then PostgreSQL for scale - WITHOUT changing application code.

```python
@runtime_checkable
class Database(Protocol):
    resource_repo: ResourceRepo
    memory_item_repo: MemoryItemRepo
    # ... same interface everywhere
```

Swap the store, keep the logic.

---

What's your go-to storage setup for AI agents? Still using Pinecone or moved to pgvector?

#ai #llm #postgres #pgvector #database
