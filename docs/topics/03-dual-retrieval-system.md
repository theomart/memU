# Topic 3: Dual Retrieval System - RAG vs LLM-Based Retrieval

## Overview

MemU offers two distinct retrieval strategies for querying stored memories: **RAG-based retrieval** (fast vector similarity search) and **LLM-based retrieval** (deep semantic understanding). Both follow the same hierarchical search pattern but differ in how they rank and filter results.

---

## The Two Retrieval Methods

| Aspect | RAG Method | LLM Method |
|--------|------------|------------|
| **Speed** | Fast (milliseconds) | Slower (seconds) |
| **Accuracy** | Good for keyword/semantic similarity | Better for complex intent |
| **Cost** | Low (embeddings only) | Higher (LLM API calls) |
| **Use Case** | High-volume, latency-sensitive | High-accuracy, complex queries |

**Configuration:** `src/memu/app/settings.py:138-164`

```python
class RetrieveConfig(BaseModel):
    method: Literal["rag", "llm"] = "rag"  # Default to fast RAG
    route_intention: bool = True  # Decide if retrieval is needed
    category: RetrieveCategoryConfig = RetrieveCategoryConfig()
    item: RetrieveItemConfig = RetrieveItemConfig()
    resource: RetrieveResourceConfig = RetrieveResourceConfig()
    sufficiency_check: bool = True  # Stop early when enough info found
```

---

## Hierarchical Search Pattern (Both Methods)

Both retrieval methods follow the same three-tier search pattern:

```
┌─────────────────────────────────────────────────────────────────┐
│  TIER 1: Categories                                             │
│  Search high-level category summaries                           │
│  → Identifies which knowledge domains are relevant              │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │ Sufficiency Check │
                    │ "Do we have enough?"│
                    └─────────┬─────────┘
                              │ (if NO)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  TIER 2: Memory Items                                           │
│  Search specific memory items within relevant categories        │
│  → Retrieves concrete facts and details                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │ Sufficiency Check │
                    │ "Do we have enough?"│
                    └─────────┬─────────┘
                              │ (if NO)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  TIER 3: Resources                                              │
│  Search original source documents/conversations                 │
│  → Provides full context and source material                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pre-Retrieval Decision (Route Intention)

Before searching, the system decides whether retrieval is needed at all.

**Code location:** `src/memu/prompts/retrieve/pre_retrieval_decision.py`

### Decision Criteria

**NO_RETRIEVE (skip retrieval):**
- Greetings, casual chat, or acknowledgments
- Questions about only the current conversation/context
- General knowledge questions
- Requests for clarification
- Meta-questions about the system itself

**RETRIEVE (search memory):**
- Questions about past events, conversations, or interactions
- Queries about user preferences, habits, or characteristics
- Requests to recall specific information
- Questions referencing historical data

### The Decision Prompt

```python
SYSTEM_PROMPT = """
# Task Objective
Determine whether the current query requires retrieving information
from memory or can be answered directly without retrieval.
If retrieval is required, rewrite the query to include relevant
contextual information.
...
"""
```

**Output format:**
```xml
<decision>
RETRIEVE or NO_RETRIEVE
</decision>

<rewritten_query>
If RETRIEVE: provide a rewritten query incorporating relevant context.
If NO_RETRIEVE: return the original query unchanged.
</rewritten_query>
```

---

## RAG-Based Retrieval (method="rag")

### How It Works

1. **Embed the query** into a vector
2. **Cosine similarity search** at each tier
3. **Sufficiency check** after each tier to potentially stop early

### Vector Search Implementation

**Code location:** `src/memu/database/inmemory/vector.py:14-49`

```python
def cosine_topk(
    query_vec: list[float],
    corpus: Iterable[tuple[str, list[float] | None]],
    k: int = 5,
) -> list[tuple[str, float]]:
    # Vectorized computation: stack all vectors into a matrix
    q = np.array(query_vec, dtype=np.float32)
    matrix = np.array(vecs, dtype=np.float32)

    # Compute all cosine similarities at once
    q_norm = np.linalg.norm(q)
    vec_norms = np.linalg.norm(matrix, axis=1)
    scores = matrix @ q / (vec_norms * q_norm + 1e-9)

    # O(n) topk selection using argpartition
    topk_indices = np.argpartition(scores, -actual_k)[-actual_k:]
    topk_indices = topk_indices[np.argsort(scores[topk_indices])[::-1]]

    return [(ids[i], float(scores[i])) for i in topk_indices]
```

**Performance optimization:** Uses NumPy's `argpartition` for O(n) top-k selection instead of O(n log n) full sort.

### RAG Workflow Steps

**Code location:** `src/memu/app/retrieve.py:106-210`

```python
def _build_rag_retrieve_workflow(self) -> list[WorkflowStep]:
    return [
        WorkflowStep(
            step_id="route_intention",
            handler=self._rag_route_intention,
            produces={"needs_retrieval", "rewritten_query"},
        ),
        WorkflowStep(
            step_id="route_category",
            handler=self._rag_route_category,
            produces={"category_hits", "query_vector"},
        ),
        WorkflowStep(
            step_id="sufficiency_after_category",
            handler=self._rag_category_sufficiency,
            produces={"proceed_to_items"},
        ),
        WorkflowStep(
            step_id="recall_items",
            handler=self._rag_recall_items,
            produces={"item_hits"},
        ),
        WorkflowStep(
            step_id="sufficiency_after_items",
            handler=self._rag_item_sufficiency,
            produces={"proceed_to_resources"},
        ),
        WorkflowStep(
            step_id="recall_resources",
            handler=self._rag_recall_resources,
            produces={"resource_hits"},
        ),
        WorkflowStep(
            step_id="build_context",
            handler=self._rag_build_context,
            produces={"response"},
        ),
    ]
```

---

## LLM-Based Retrieval (method="llm")

### How It Works

1. **Present all candidates** to the LLM as text
2. **LLM ranks by relevance** using semantic understanding
3. **Sufficiency check** after each tier

### LLM Ranking Prompts

**Category Ranker:** `src/memu/prompts/retrieve/llm_category_ranker.py`
**Item Ranker:** `src/memu/prompts/retrieve/llm_item_ranker.py`
**Resource Ranker:** `src/memu/prompts/retrieve/llm_resource_ranker.py`

### Example: LLM Category Ranking

```python
# From src/memu/app/retrieve.py:1176-1199
async def _llm_rank_categories(self, query, top_k, ctx, store, llm_client):
    categories_data = self._format_categories_for_llm(store)
    prompt = LLM_CATEGORY_RANKER_PROMPT.format(
        query=query,
        top_k=top_k,
        categories_data=categories_data,
    )
    llm_response = await client.summarize(prompt)
    return self._parse_llm_category_response(llm_response, store)
```

### LLM Workflow Steps

**Code location:** `src/memu/app/retrieve.py:430-512`

The LLM workflow mirrors the RAG workflow but uses LLM ranking instead of vector similarity:

```python
def _build_llm_retrieve_workflow(self) -> list[WorkflowStep]:
    return [
        WorkflowStep(step_id="route_intention", ...),
        WorkflowStep(
            step_id="route_category",
            handler=self._llm_route_category,  # LLM-based
            capabilities={"llm"},
        ),
        WorkflowStep(step_id="sufficiency_after_category", ...),
        WorkflowStep(
            step_id="recall_items",
            handler=self._llm_recall_items,  # LLM-based
            capabilities={"llm"},
        ),
        # ... similar pattern
    ]
```

---

## Sufficiency Check: The "Smart Stop"

A key innovation in MemU's retrieval is the **sufficiency check** - the system asks the LLM "Do we have enough information to answer this query?" after each tier.

### Why This Matters

1. **Reduces latency** - Stop searching when you have enough
2. **Reduces noise** - Avoid retrieving irrelevant lower-tier results
3. **Query rewriting** - If more info needed, rewrite the query to be more specific

### The Sufficiency Check Prompt

**Code location:** `src/memu/app/retrieve.py:706-744`

```python
async def _decide_if_retrieval_needed(
    self,
    query: str,
    context_queries: list[dict],
    retrieved_content: str | None = None,
):
    # Build prompt with retrieved content
    prompt = PRE_RETRIEVAL_USER_PROMPT.format(
        query=query,
        conversation_history=history_text,
        retrieved_content=content_text,
    )

    response = await client.summarize(prompt, system_prompt=sys_prompt)
    decision = self._extract_decision(response)  # RETRIEVE or NO_RETRIEVE
    rewritten = self._extract_rewritten_query(response)

    return decision == "RETRIEVE", rewritten
```

---

## Query Rewriting for Context Resolution

Queries often contain pronouns or references that need context to understand:

**Example:**
- User asks: "What did she say about that?"
- Rewritten: "What did Sarah say about the project deadline?"

This is handled at each tier transition, incorporating:
- Conversation history
- Previously retrieved content
- Original query context

---

## Configuration Options

### Per-Tier Configuration

```python
class RetrieveCategoryConfig(BaseModel):
    enabled: bool = True  # Enable/disable tier
    top_k: int = 5        # Max results per tier

class RetrieveConfig(BaseModel):
    category: RetrieveCategoryConfig
    item: RetrieveItemConfig
    resource: RetrieveResourceConfig
    sufficiency_check: bool = True
    sufficiency_check_llm_profile: str = "default"
    llm_ranking_llm_profile: str = "default"
```

### LLM Profile Assignment

Different LLM models can be used for different operations:

```python
config = RetrieveConfig(
    method="llm",
    sufficiency_check_llm_profile="fast-model",  # Cheaper model for checks
    llm_ranking_llm_profile="smart-model",       # Better model for ranking
)
```

---

## Response Structure

Both methods return the same response structure:

```python
{
    "needs_retrieval": bool,
    "original_query": str,
    "rewritten_query": str,
    "next_step_query": str | None,
    "categories": [
        {"id": "...", "name": "...", "summary": "...", "score": 0.85}
    ],
    "items": [
        {"id": "...", "memory_type": "profile", "summary": "...", "score": 0.92}
    ],
    "resources": [
        {"id": "...", "url": "...", "caption": "...", "score": 0.78}
    ]
}
```

---

## Code References

| Component | File | Line |
|-----------|------|------|
| Retrieve Mixin | `src/memu/app/retrieve.py` | 27-1379 |
| RAG Workflow | `src/memu/app/retrieve.py` | 106-210 |
| LLM Workflow | `src/memu/app/retrieve.py` | 430-512 |
| Cosine TopK | `src/memu/database/inmemory/vector.py` | 14-49 |
| Pre-Retrieval Decision | `src/memu/prompts/retrieve/pre_retrieval_decision.py` | Full file |
| Sufficiency Check | `src/memu/app/retrieve.py` | 706-744 |
| RetrieveConfig | `src/memu/app/settings.py` | 138-164 |

---

## GitHub Discussion Context

- The dual retrieval system supports both cost-conscious (RAG) and accuracy-focused (LLM) use cases
- **PR #218** (Event-Driven Orchestration) could enable async retrieval for even faster response times
- The sufficiency check is a key differentiator from simple RAG systems

---

*This document is intended for technical copywriters explaining MemU's retrieval system to AI engineers and system architects.*
