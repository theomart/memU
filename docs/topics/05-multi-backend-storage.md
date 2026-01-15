# Topic 5: Multi-Backend Storage System

## Overview

MemU implements a **pluggable storage architecture** that supports three database backends: **InMemory**, **SQLite**, and **PostgreSQL**. Each backend implements the same `Database` protocol, enabling seamless switching between development, lightweight, and production deployments.

---

## The Database Protocol

All storage backends implement this protocol, ensuring consistent behavior regardless of the underlying storage technology.

**Code location:** `src/memu/database/interfaces.py:12-26`

```python
@runtime_checkable
class Database(Protocol):
    """Backend-agnostic database contract."""

    resource_repo: ResourceRepo
    memory_category_repo: MemoryCategoryRepo
    memory_item_repo: MemoryItemRepo
    category_item_repo: CategoryItemRepo

    resources: dict[str, ResourceRecord]
    items: dict[str, MemoryItemRecord]
    categories: dict[str, MemoryCategoryRecord]
    relations: list[CategoryItemRecord]

    def close(self) -> None: ...
```

### Why Protocol-Based Design?

1. **Duck typing** - Any object that implements these methods works
2. **No inheritance required** - Clean composition over inheritance
3. **Runtime checking** - `@runtime_checkable` enables `isinstance()` checks
4. **Easy testing** - Mock implementations are trivial to create

---

## Backend 1: InMemory Store

**Best for:** Development, testing, ephemeral sessions, prototyping.

**Code location:** `src/memu/database/inmemory/repo.py:20-60`

```python
class InMemoryStore(Database):
    def __init__(
        self,
        *,
        scope_model: type[BaseModel] | None = None,
        state: InMemoryState | None = None,
    ) -> None:
        self.scope_model = scope_model or BaseModel
        self.state = state or InMemoryState()

        # Direct dict references for fast access
        self.resources: dict[str, Resource] = self.state.resources
        self.items: dict[str, MemoryItem] = self.state.items
        self.categories: dict[str, MemoryCategory] = self.state.categories
        self.relations: list[CategoryItem] = self.state.relations

        # Initialize repositories
        self.resource_repo = InMemoryResourceRepository(state=self.state)
        self.memory_category_repo = InMemoryMemoryCategoryRepository(state=self.state)
        self.memory_item_repo = InMemoryMemoryItemRepository(state=self.state)
        self.category_item_repo = InMemoryCategoryItemRepository(state=self.state)
```

### Characteristics

| Aspect | Details |
|--------|---------|
| **Persistence** | None - data lost on restart |
| **Performance** | Fastest - pure Python dicts |
| **Vector Search** | Custom NumPy cosine similarity |
| **Setup** | Zero configuration |
| **Memory Usage** | Grows with data |

### Vector Search Implementation

**Code location:** `src/memu/database/inmemory/vector.py`

```python
def cosine_topk(
    query_vec: list[float],
    corpus: Iterable[tuple[str, list[float] | None]],
    k: int = 5,
) -> list[tuple[str, float]]:
    # Vectorized NumPy computation
    q = np.array(query_vec, dtype=np.float32)
    matrix = np.array(vecs, dtype=np.float32)

    # Batch cosine similarity
    q_norm = np.linalg.norm(q)
    vec_norms = np.linalg.norm(matrix, axis=1)
    scores = matrix @ q / (vec_norms * q_norm + 1e-9)

    # O(n) top-k selection
    topk_indices = np.argpartition(scores, -actual_k)[-actual_k:]
    return [(ids[i], float(scores[i])) for i in topk_indices]
```

---

## Backend 2: SQLite Store

**Best for:** Lightweight deployments, file-based persistence, single-user applications.

**Code location:** `src/memu/database/sqlite/sqlite.py:25-146`

```python
class SQLiteStore(Database):
    """SQLite database store implementation.

    This store provides a lightweight, file-based database backend for MemU.
    It uses SQLite for metadata storage and brute-force cosine similarity
    for vector search (native vector support is not available in SQLite).
    """

    def __init__(
        self,
        *,
        dsn: str,  # e.g., "sqlite:///path/to/db.sqlite"
        scope_model: type[BaseModel] | None = None,
    ) -> None:
        self.dsn = dsn
        self._scope_model = scope_model or BaseModel
        self._sessions = SQLiteSessionManager(dsn=self.dsn)
        self._sqla_models = get_sqlite_sqlalchemy_models(scope_model=self._scope_model)

        # Create tables automatically
        self._create_tables()

        # Initialize repositories
        self.resource_repo = SQLiteResourceRepo(...)
        self.memory_category_repo = SQLiteMemoryCategoryRepo(...)
        self.memory_item_repo = SQLiteMemoryItemRepo(...)
        self.category_item_repo = SQLiteCategoryItemRepo(...)
```

### Characteristics

| Aspect | Details |
|--------|---------|
| **Persistence** | File-based - survives restarts |
| **Performance** | Good - native SQL optimization |
| **Vector Search** | Brute-force cosine (no native support) |
| **Setup** | Single DSN string |
| **Portability** | Excellent - single file |

### Configuration

```python
from memu.database.sqlite import SQLiteStore

store = SQLiteStore(
    dsn="sqlite:///./data/memu.sqlite",
    scope_model=MyUserScope,
)
```

### Table Schema

The SQLite backend creates these tables:
- `sqlite_resources`
- `sqlite_memory_items`
- `sqlite_memory_categories`
- `sqlite_category_items`

Tables are created automatically via SQLModel/SQLAlchemy.

---

## Backend 3: PostgreSQL Store

**Best for:** Production deployments, high concurrency, native vector operations.

**Code location:** `src/memu/database/postgres/postgres.py:23-109`

```python
class PostgresStore(Database):
    def __init__(
        self,
        *,
        dsn: str,
        ddl_mode: DDLMode = "create",
        vector_provider: str | None = None,  # "pgvector" for native vectors
        scope_model: type[BaseModel] | None = None,
    ) -> None:
        require_sqlalchemy()
        self.dsn = dsn
        self.ddl_mode = ddl_mode
        self._use_vector_type = vector_provider == "pgvector"
        self._sessions = SessionManager(dsn=self.dsn)
        self._sqla_models = get_sqlalchemy_models(scope_model=self._scope_model)

        # Run migrations
        run_migrations(dsn=self.dsn, scope_model=self._scope_model, ddl_mode=self.ddl_mode)

        # Initialize repositories with pgvector support
        self.memory_item_repo = PostgresMemoryItemRepo(
            ...,
            use_vector=self._use_vector_type,  # Enable native vector ops
        )
```

### Characteristics

| Aspect | Details |
|--------|---------|
| **Persistence** | Full ACID compliance |
| **Performance** | Excellent - native indexing |
| **Vector Search** | Native pgvector support |
| **Setup** | Requires PostgreSQL server |
| **Scalability** | Production-grade |

### pgvector Integration

When `vector_provider="pgvector"` is set:

1. **Native vector type** - Embeddings stored as `vector(n)` columns
2. **HNSW indexing** - Approximate nearest neighbor search
3. **Hardware acceleration** - SIMD operations on modern CPUs

```python
from memu.database.postgres import PostgresStore

store = PostgresStore(
    dsn="postgresql://user:pass@localhost:5432/memu",
    vector_provider="pgvector",  # Enable native vectors
    ddl_mode="create",
)
```

### DDL Modes

| Mode | Behavior |
|------|----------|
| `create` | Create tables if they don't exist |
| `validate` | Verify tables exist, error if not |

---

## Configuration via Settings

**Code location:** `src/memu/app/settings.py:250-272`

```python
class MetadataStoreConfig(BaseModel):
    provider: Literal["inmemory", "postgres", "sqlite"] = "inmemory"
    ddl_mode: Literal["create", "validate"] = "create"
    dsn: str | None = None  # Required for postgres/sqlite

class VectorIndexConfig(BaseModel):
    provider: Literal["bruteforce", "pgvector", "none"] = "bruteforce"
    dsn: str | None = None  # For pgvector

class DatabaseConfig(BaseModel):
    metadata_store: MetadataStoreConfig
    vector_index: VectorIndexConfig | None = None
```

### Example Configurations

**Development (InMemory):**
```python
config = DatabaseConfig(
    metadata_store=MetadataStoreConfig(provider="inmemory")
)
```

**Lightweight (SQLite):**
```python
config = DatabaseConfig(
    metadata_store=MetadataStoreConfig(
        provider="sqlite",
        dsn="sqlite:///./data/memu.sqlite"
    )
)
```

**Production (PostgreSQL + pgvector):**
```python
config = DatabaseConfig(
    metadata_store=MetadataStoreConfig(
        provider="postgres",
        dsn="postgresql://user:pass@db.example.com:5432/memu"
    ),
    vector_index=VectorIndexConfig(
        provider="pgvector",
        dsn="postgresql://user:pass@db.example.com:5432/memu"
    )
)
```

---

## Multi-Tenant Support (Scope Models)

All backends support **scope models** for multi-tenant isolation.

**Code location:** `src/memu/database/models.py:48-61`

```python
def merge_scope_model(
    scope_model: type[BaseModel],
    base_model: type[T],
) -> type[T]:
    """Dynamically inject scope fields into a record model."""
    scope_fields = {}
    for field_name, field_info in scope_model.model_fields.items():
        scope_fields[field_name] = (field_info.annotation | None, None)

    return create_model(
        f"Scoped{base_model.__name__}",
        __base__=base_model,
        **scope_fields,
    )
```

### Scope Model Example

```python
from pydantic import BaseModel

class MyScope(BaseModel):
    user_id: str
    org_id: str | None = None

# All records now have user_id and org_id fields
store = InMemoryStore(scope_model=MyScope)

# Query with scope filtering
items = store.memory_item_repo.list_items(
    where={"user_id": "user_123", "org_id": "org_456"}
)
```

---

## Repository Pattern

Each storage type (Resource, MemoryItem, MemoryCategory, CategoryItem) has its own repository interface.

**Code location:** `src/memu/database/repositories.py`

```python
class MemoryItemRepo(Protocol):
    def create(self, item: MemoryItem) -> MemoryItem: ...
    def get(self, item_id: str) -> MemoryItem | None: ...
    def list_items(self, where: dict | None = None) -> dict[str, MemoryItem]: ...
    def vector_search_items(
        self,
        query_vec: list[float],
        top_k: int,
        where: dict | None = None,
    ) -> list[tuple[str, float]]: ...
    def delete(self, item_id: str) -> bool: ...
```

### Implementation Hierarchy

```
Protocol (Interface)
    ├── InMemory Implementation
    │   └── Pure Python dicts + NumPy
    ├── SQLite Implementation
    │   └── SQLAlchemy + brute-force vectors
    └── PostgreSQL Implementation
        └── SQLAlchemy + optional pgvector
```

---

## Migration Support

The PostgreSQL backend includes Alembic-based migration support.

**Code location:** `src/memu/database/postgres/migration.py`

```python
def run_migrations(
    dsn: str,
    scope_model: type[BaseModel],
    ddl_mode: DDLMode = "create",
) -> None:
    """Run database migrations based on DDL mode."""
    # Creates tables, handles schema changes
    ...
```

---

## Performance Comparison

| Operation | InMemory | SQLite | PostgreSQL | PostgreSQL+pgvector |
|-----------|----------|--------|------------|---------------------|
| Insert | ~0.01ms | ~1ms | ~2ms | ~2ms |
| Get by ID | ~0.01ms | ~0.5ms | ~1ms | ~1ms |
| List (100) | ~0.1ms | ~5ms | ~10ms | ~10ms |
| Vector Search (10k) | ~50ms | ~100ms | ~150ms | ~5ms |

*Note: Vector search with pgvector uses HNSW approximate search, dramatically faster at scale.*

---

## Code References

| Component | File | Line |
|-----------|------|------|
| Database Protocol | `src/memu/database/interfaces.py` | 12-26 |
| InMemoryStore | `src/memu/database/inmemory/repo.py` | 20-60 |
| SQLiteStore | `src/memu/database/sqlite/sqlite.py` | 25-146 |
| PostgresStore | `src/memu/database/postgres/postgres.py` | 23-109 |
| Vector Search (NumPy) | `src/memu/database/inmemory/vector.py` | 14-49 |
| Scope Model Merge | `src/memu/database/models.py` | 48-61 |
| DatabaseConfig | `src/memu/app/settings.py` | 250-272 |

---

## GitHub Discussion Context

- **Issue #19** (Docker-compose issues) - Often related to PostgreSQL connection setup
- **PR #232** (Nullable resource_id fix) - Fixed a bug in PostgreSQL item handling
- The storage architecture enables the clear memory feature in **PR #239**

---

*This document is intended for technical copywriters explaining MemU's storage architecture to DevOps engineers and database administrators.*
