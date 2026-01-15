# Topic 2: Memory Type Extraction with LLM Prompts

## Overview

MemU uses sophisticated **prompt engineering** to extract structured memories from unstructured conversations. The system employs a **modular prompt block architecture** that allows customization while maintaining consistency across the 5 memory types.

---

## The 5 Memory Types

Each memory type captures a different aspect of user information:

| Memory Type | What It Captures | Temporal Aspect |
|-------------|------------------|-----------------|
| **Profile** | User characteristics, preferences, stable traits | Long-term stable |
| **Event** | Specific experiences at particular times | Time-bound |
| **Knowledge** | Factual information and concepts | Accumulated |
| **Behavior** | Patterns, habits, routines | Recurring |
| **Skill** | Demonstrated techniques and capabilities | Demonstrated |

**Default active types:** By default, MemU extracts only `profile` and `event` types for efficiency.

**Code location:** `src/memu/prompts/memory_type/__init__.py:4`

```python
DEFAULT_MEMORY_TYPES: list[str] = ["profile", "event"]
```

---

## Prompt Block Architecture

MemU uses a **modular prompt system** where each extraction prompt is composed of reusable blocks. This enables:

1. **Consistency** across memory types
2. **Easy customization** of individual sections
3. **A/B testing** of prompt variations

### The 7 Prompt Blocks

| Block | Purpose | Default Ordinal |
|-------|---------|-----------------|
| `objective` | Task definition and role | 10 |
| `workflow` | Step-by-step extraction process | 20 |
| `rules` | Extraction constraints and requirements | 30 |
| `category` | Memory category definitions | 40 |
| `output` | Output format specification | 50 |
| `examples` | Few-shot examples | 60 |
| `input` | The actual resource content | 90 |

**Code location:** `src/memu/prompts/memory_type/__init__.py:28-36`

```python
DEFAULT_MEMORY_CUSTOM_PROMPT_ORDINAL: dict[str, int] = {
    "objective": 10,
    "workflow": 20,
    "rules": 30,
    "category": 40,
    "output": 50,
    "examples": 60,
    "input": 90,
}
```

---

## Deep Dive: Profile Extraction Prompt

The profile extraction prompt (`src/memu/prompts/memory_type/profile.py`) demonstrates the full prompt structure.

### Block 1: Objective

```
# Task Objective
You are a professional User Memory Extractor. Your core task is to extract
independent user memory items about the user (e.g., basic info, preferences,
habits, other long-term stable traits).
```

**Purpose:** Establishes the LLM's role and primary goal.

### Block 2: Workflow

```
# Workflow
Read the full conversation to understand topics and meanings.
## Extract memories
Select turns that contain valuable User Information and extract user info memory items.
## Review & validate
Merge semantically similar items.
Resolve contradictions by keeping the latest / most certain item.
## Final output
Output User Information.
```

**Purpose:** Provides a step-by-step process for the LLM to follow.

### Block 3: Rules

The rules block is the most critical section, containing:

**General requirements (must satisfy all):**
- Use "user" to refer to the user consistently
- Each memory item must be complete and self-contained
- Each memory item must express one single complete piece of information
- Similar/redundant items must be merged into one
- Each memory item must be < 30 words

**Special rules for profile extraction:**
- Any event-related item is forbidden in User Information
- Do not extract content obtained only through model's follow-up questions

**Forbidden content:**
- Knowledge Q&A without a clear user fact
- Trivial updates that do not add meaningful value
- Turns where the user did not respond
- Illegal/harmful sensitive topics
- Private financial accounts, IDs, addresses
- Content mentioned only by the assistant

**Code location:** `src/memu/prompts/memory_type/profile.py:73-102`

### Block 4: Output Format

MemU uses XML format for structured extraction:

```xml
<item>
    <memory>
        <content>User memory item content 1</content>
        <categories>
            <category>Category Name</category>
        </categories>
    </memory>
    <memory>
        <content>User memory item content 2</content>
        <categories>
            <category>Category Name</category>
        </categories>
    </memory>
</item>
```

**Why XML?** XML provides clear structure while being more forgiving of formatting variations than JSON.

### Block 5: Examples (Few-Shot Learning)

The prompt includes input/output examples with explanations:

**Input:**
```
user: Hi, are you busy? I just got off work and I'm going to the supermarket.
assistant: Not busy. Are you cooking for yourself?
user: Yes. It's healthier. I work as a product manager in an internet company.
      I'm 30 this year. After work I like experimenting with cooking.
```

**Output:**
```xml
<item>
    <memory>
        <content>The user works as a product manager at an internet company</content>
        <categories><category>Basic Information</category></categories>
    </memory>
    <memory>
        <content>The user is 30 years old</content>
        <categories><category>Basic Information</category></categories>
    </memory>
</item>
```

**Explanation:**
> Only stable user facts explicitly stated by the user are extracted.
> The travel plan and packing annoyance are events/temporary states, so they are not extracted as User Information.

---

## Customizing Prompts

### Per-Memory-Type Customization

Each memory type has its own prompt module:
- `src/memu/prompts/memory_type/profile.py`
- `src/memu/prompts/memory_type/event.py`
- `src/memu/prompts/memory_type/knowledge.py`
- `src/memu/prompts/memory_type/behavior.py`
- `src/memu/prompts/memory_type/skill.py`

### Configuration-Based Customization

Override prompts via `MemorizeConfig`:

```python
from memu.app.settings import MemorizeConfig, CustomPrompt, PromptBlock

config = MemorizeConfig(
    memory_types=["profile", "event", "skill"],  # Custom type selection
    memory_type_prompts={
        "profile": CustomPrompt({
            "objective": PromptBlock(
                ordinal=10,
                prompt="Custom objective for profile extraction..."
            ),
            "rules": PromptBlock(
                ordinal=30,
                prompt="Custom rules for this use case..."
            ),
        })
    }
)
```

**Code location:** `src/memu/app/settings.py:167-196`

---

## The Extraction Flow

### Step 1: Build Prompt

```python
# From src/memu/app/memorize.py:947-961
def _build_memory_type_prompt(self, *, memory_type, resource_text, categories_str):
    configured_prompt = self.memorize_config.memory_type_prompts.get(memory_type)
    if configured_prompt is None:
        template = MEMORY_TYPE_PROMPTS.get(memory_type)
    elif isinstance(configured_prompt, str):
        template = configured_prompt
    else:
        template = self._resolve_custom_prompt(
            configured_prompt, MEMORY_TYPE_CUSTOM_PROMPTS.get(memory_type)
        )
    return template.format(resource=safe_resource, categories_str=safe_categories)
```

### Step 2: Parallel LLM Calls

For efficiency, extraction for all memory types happens in parallel:

```python
# From src/memu/app/memorize.py:516-527
prompts = [
    self._build_memory_type_prompt(
        memory_type=mtype,
        resource_text=resource_text,
        categories_str=categories_prompt_str,
    )
    for mtype in memory_types
]
tasks = [client.summarize(prompt_text) for prompt_text in valid_prompts]
responses = await asyncio.gather(*tasks)
```

### Step 3: Parse XML Response

```python
# From src/memu/app/memorize.py:1166-1206
def _parse_memory_type_response_xml(self, raw: str) -> list[dict[str, Any]]:
    root = ET.fromstring(xml_content)
    result = []
    for memory_elem in root.findall("memory"):
        content_elem = memory_elem.find("content")
        categories_elem = memory_elem.find("categories")
        # ... parse and validate
    return result
```

---

## Key Design Decisions

### 1. XML Over JSON

**Why:** LLMs sometimes struggle with JSON escaping and bracket matching. XML is more forgiving and the hierarchical structure maps well to the memory data model.

### 2. Explicit Forbidden Content

**Why:** Without explicit exclusion rules, LLMs tend to over-extract, including assistant suggestions as user facts.

### 3. Word Limit per Memory

**Why:** The 30-word limit encourages atomic, reusable memory items that can be combined differently in various contexts.

### 4. Category Assignment in Extraction

**Why:** Having the LLM assign categories during extraction (rather than as a separate step) reduces hallucination and ensures semantic coherence.

---

## Code References

| Component | File | Line |
|-----------|------|------|
| Memory Types Init | `src/memu/prompts/memory_type/__init__.py` | 1-45 |
| Profile Prompt | `src/memu/prompts/memory_type/profile.py` | 1-191 |
| Event Prompt | `src/memu/prompts/memory_type/event.py` | Full file |
| Prompt Building | `src/memu/app/memorize.py` | 947-961 |
| XML Parsing | `src/memu/app/memorize.py` | 1166-1206 |
| Parallel Extraction | `src/memu/app/memorize.py` | 505-527 |

---

## GitHub Discussion Context

- **PR #135** (Expanded config) - Modified prompt configuration options
- **PR #108** (Add usecase example) - Added skills memory extraction examples
- The prompt system directly supports the "2026 New Year Challenge" tracks that focus on memory extraction improvements

---

*This document is intended for technical copywriters explaining MemU's LLM-based memory extraction to ML engineers and AI practitioners.*
