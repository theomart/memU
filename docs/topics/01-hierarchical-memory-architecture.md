# Topic 1: Hierarchical Memory Architecture

## Overview

MemU implements a **three-layer hierarchical memory system** that organizes AI agent memories from raw inputs down to organized, queryable summaries. This architecture enables both granular memory recall and high-level semantic understanding.

---

## The Three Layers Explained

### Layer 1: Resources (Raw Input Layer)

**What it is:** The foundational layer that stores raw multimodal inputs - the original conversations, documents, images, videos, and audio files that an AI agent encounters.

**Key properties:**
- **URL**: Reference to the original source
- **Modality**: Type of content (conversation, document, image, video, audio)
- **Local Path**: Where the file is stored locally
- **Caption**: A generated summary/description of the content
- **Embedding**: Vector representation for semantic search

**Code location:** `src/memu/database/models.py:21-27`

```python
class Resource(BaseRecord):
    url: str
    modality: str
    local_path: str
    caption: str | None = None
    embedding: list[float] | None = None
```

**Why this matters:** Resources serve as the ground truth - the original source material. When an AI agent needs to trace back where a memory came from, it references the Resource layer.

---

### Layer 2: Memory Items (Extracted Facts Layer)

**What it is:** Discrete, atomic pieces of information extracted from Resources. Each MemoryItem represents a single fact, preference, or piece of knowledge about the user.

**Key properties:**
- **Resource ID**: Links back to the source Resource
- **Memory Type**: One of 5 types (profile, event, knowledge, behavior, skill)
- **Summary**: The actual extracted information as text
- **Embedding**: Vector representation for semantic search

**Code location:** `src/memu/database/models.py:29-33`

```python
class MemoryItem(BaseRecord):
    resource_id: str | None
    memory_type: MemoryType
    summary: str
    embedding: list[float] | None = None
```

**The 5 Memory Types:**

| Type | Description | Example |
|------|-------------|---------|
| **Profile** | User characteristics, preferences, stable traits | "The user is 30 years old and works as a product manager" |
| **Event** | Specific experiences at particular times | "The user went hiking with family last weekend" |
| **Knowledge** | Factual information and concepts | "The user knows Python and has 5 years of experience" |
| **Behavior** | Patterns, habits, routines | "The user prefers working in the morning" |
| **Skill** | Demonstrated techniques and capabilities | "The user is proficient at data visualization" |

---

### Layer 3: Memory Categories (Aggregated Summary Layer)

**What it is:** High-level organizational buckets that group related memory items and maintain progressive summaries. Categories provide a bird's-eye view of what the system knows about a user.

**Key properties:**
- **Name**: Category identifier (e.g., "personal_info", "preferences")
- **Description**: What this category captures
- **Summary**: Auto-generated progressive summary of all items in this category
- **Embedding**: Vector representation for semantic search

**Code location:** `src/memu/database/models.py:36-40`

```python
class MemoryCategory(BaseRecord):
    name: str
    description: str
    embedding: list[float] | None = None
    summary: str | None = None
```

**Default Categories (from `src/memu/app/settings.py:74-89`):**
- personal_info
- preferences
- relationships
- activities
- goals
- experiences
- knowledge
- opinions
- habits
- work_life

---

## The CategoryItem Relation

**What it is:** A many-to-many join table that links Memory Items to Categories. One memory item can belong to multiple categories.

**Code location:** `src/memu/database/models.py:43-45`

```python
class CategoryItem(BaseRecord):
    item_id: str
    category_id: str
```

**Example:** A memory item like "The user enjoys hiking with family on weekends" could be linked to:
- `activities` (hiking)
- `relationships` (family)
- `habits` (weekend routine)

---

## Data Flow: From Input to Organized Memory

```
┌──────────────────────────────────────────────────────────────────┐
│  STEP 1: Ingest Resource                                         │
│  Raw conversation/document/image enters the system               │
│  → Stored as a Resource record with URL, modality, local path    │
└──────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│  STEP 2: Preprocess Multimodal                                   │
│  Convert to text format for LLM processing                       │
│  → Images → vision model description                             │
│  → Audio → transcription                                         │
│  → Video → frame extraction + vision                             │
└──────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│  STEP 3: Extract Memory Items                                    │
│  LLM analyzes text and extracts discrete memories                │
│  → Each memory typed (profile/event/knowledge/behavior/skill)    │
│  → Each memory gets an embedding vector                          │
└──────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│  STEP 4: Categorize Items                                        │
│  Link memory items to appropriate categories                     │
│  → Vector similarity matches items to category embeddings        │
│  → Creates CategoryItem relation records                         │
└──────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│  STEP 5: Update Category Summaries                               │
│  LLM updates category summaries with new information             │
│  → Progressive summarization keeps summaries current             │
│  → Maintains manageable summary lengths                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Architectural Benefits

### 1. Traceability
Every piece of information can be traced back to its source. Memory items link to resources, categories link to items. This creates an audit trail for AI agent knowledge.

### 2. Granularity Control
- Need high-level context? Query category summaries
- Need specific details? Query memory items
- Need original source? Access resources

### 3. Progressive Summarization
Category summaries grow intelligently. As new memory items are added, the LLM updates summaries to incorporate new information while maintaining coherence.

### 4. Multi-Tenant Support
The `merge_scope_model` function (`src/memu/database/models.py:48-61`) allows adding custom scope fields (like `user_id`, `org_id`) to all record types, enabling memory isolation between different users or organizations.

---

## Code References

| Component | File | Line |
|-----------|------|------|
| Memory Types Definition | `src/memu/database/models.py` | 10 |
| Resource Model | `src/memu/database/models.py` | 21-27 |
| MemoryItem Model | `src/memu/database/models.py` | 29-33 |
| MemoryCategory Model | `src/memu/database/models.py` | 36-40 |
| CategoryItem Model | `src/memu/database/models.py` | 43-45 |
| Default Categories | `src/memu/app/settings.py` | 74-89 |
| Memorize Workflow | `src/memu/app/memorize.py` | 97-166 |

---

## GitHub Discussion Context

The hierarchical architecture directly supports features discussed in the community:

- **PR #129** (WorkflowMixin refactoring) - Aims to clean up the workflow code that manages the memorization pipeline
- **PR #240** (Workflow step interceptor) - Enables hooking into each step of the memorize workflow
- **Issue #19** - Docker setup issues often relate to ensuring the blob storage paths are correctly configured for Resources

---

*This document is intended for technical copywriters explaining MemU's memory architecture to developers and AI engineers.*
