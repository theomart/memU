# Topic 4: Workflow Step Interceptors - Extensibility Through Hooks

## Overview

MemU's **Workflow Interceptor System** provides hooks that execute before, after, or on error of each workflow step. This enables observability, custom logging, validation, and extension of the memory processing pipeline without modifying core code.

---

## The Interceptor Pattern

Workflow interceptors follow the **Aspect-Oriented Programming (AOP)** pattern, allowing cross-cutting concerns to be separated from business logic.

```
┌─────────────────────────────────────────────────────────────────┐
│  BEFORE INTERCEPTORS                                            │
│  → Logging, validation, authentication, timing start            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  WORKFLOW STEP EXECUTION                                        │
│  → Core business logic (memorize, retrieve, etc.)               │
└─────────────────────────────────────────────────────────────────┘
                              │
               ┌──────────────┼──────────────┐
               ▼              │              ▼
     ┌─────────────────┐      │     ┌─────────────────┐
     │ AFTER (success) │      │     │ ON ERROR        │
     │ → Logging,      │      │     │ → Error logging │
     │   metrics,      │      │     │   recovery,     │
     │   notifications │      │     │   alerts        │
     └─────────────────┘      │     └─────────────────┘
                              │
                              ▼
                    [Next Step or Return]
```

---

## The WorkflowInterceptorRegistry

The core component that manages all registered interceptors.

**Code location:** `src/memu/workflow/interceptor.py:56-166`

```python
class WorkflowInterceptorRegistry:
    """
    Registry for workflow step interceptors.

    Interceptors are called before and after each workflow step execution.
    Unlike LLM interceptors, workflow interceptors do not support filtering,
    priority, or ordering - they are called in registration order.
    """

    def __init__(self, *, strict: bool = False) -> None:
        self._before: tuple[_WorkflowInterceptor, ...] = ()
        self._after: tuple[_WorkflowInterceptor, ...] = ()
        self._on_error: tuple[_WorkflowInterceptor, ...] = ()
        self._lock = threading.Lock()
        self._seq = 0
        self._strict = strict  # If True, exceptions propagate
```

### Key Design Decisions

1. **Thread-safe registration** via `threading.Lock()`
2. **Immutable tuples** for interceptor lists (safe for concurrent reads)
3. **Strict mode** option to control exception propagation
4. **Sequence-based IDs** for disposal handling

---

## Three Types of Interceptors

### 1. Before Interceptors

Execute before each workflow step begins.

**Use cases:**
- Input validation
- Authentication/authorization checks
- Start timing measurements
- Logging step initiation

```python
# Registration via MemoryService
service.intercept_before_workflow_step(
    fn=my_before_handler,
    name="validation-check"
)
```

**Handler signature:**
```python
def my_before_handler(step_context: WorkflowStepContext, state: WorkflowState):
    print(f"Starting step: {step_context.step_id}")
    # Validate state, log, etc.
```

### 2. After Interceptors

Execute after each workflow step completes successfully. Called in **reverse registration order** (LIFO).

**Use cases:**
- Log completion
- Calculate duration
- Send notifications
- Cache results

```python
service.intercept_after_workflow_step(
    fn=my_after_handler,
    name="completion-logger"
)
```

**Handler signature:**
```python
def my_after_handler(step_context: WorkflowStepContext, state: WorkflowState):
    print(f"Completed step: {step_context.step_id}")
    # Log results, update metrics
```

### 3. On Error Interceptors

Execute when a workflow step raises an exception. Called in **reverse registration order** (LIFO).

**Use cases:**
- Error logging
- Alert notifications
- Recovery attempts
- State cleanup

```python
service.intercept_on_error_workflow_step(
    fn=my_error_handler,
    name="error-alert"
)
```

**Handler signature:**
```python
def my_error_handler(
    step_context: WorkflowStepContext,
    state: WorkflowState,
    error: Exception
):
    print(f"Step {step_context.step_id} failed: {error}")
    # Send alert, log error details
```

---

## WorkflowStepContext

Each interceptor receives context about the current step.

**Code location:** `src/memu/workflow/interceptor.py:16-24`

```python
@dataclass(frozen=True)
class WorkflowStepContext:
    """Context information for a workflow step execution."""

    workflow_name: str    # "memorize", "retrieve_rag", "retrieve_llm"
    step_id: str          # "ingest_resource", "extract_items", etc.
    step_role: str        # "ingest", "extract", "persist", etc.
    step_context: dict    # Additional metadata
```

### Available Context Values

| Field | Example | Purpose |
|-------|---------|---------|
| `workflow_name` | "memorize" | Identify which pipeline is running |
| `step_id` | "extract_items" | Identify the specific step |
| `step_role` | "extract" | Understand the step's purpose |
| `step_context` | `{"llm_profile": "default"}` | Step-specific configuration |

---

## Interceptor Handle: Safe Disposal

When you register an interceptor, you receive a **handle** that allows clean removal.

**Code location:** `src/memu/workflow/interceptor.py:40-53`

```python
class WorkflowInterceptorHandle:
    """Handle for disposing a registered workflow interceptor."""

    def dispose(self) -> bool:
        """Remove the interceptor from the registry. Returns True if removed."""
        if self._disposed:
            return False
        self._disposed = True
        return self._registry.remove(self._interceptor_id)
```

### Usage Pattern

```python
# Register and keep handle
handle = service.intercept_before_workflow_step(
    fn=temporary_logger,
    name="temp-debug"
)

# ... do work ...

# Clean up when done
handle.dispose()
```

---

## Async Support

Interceptors can be either synchronous or asynchronous. The system automatically awaits async handlers.

**Code location:** `src/memu/workflow/interceptor.py:205-218`

```python
async def _safe_invoke_interceptor(
    interceptor: _WorkflowInterceptor,
    strict: bool,
    *args: Any,
) -> None:
    try:
        result = interceptor.fn(*args)
        if inspect.isawaitable(result):
            await result  # Handle async functions
    except Exception:
        if strict:
            raise
        logger.exception("Workflow interceptor failed: %s", interceptor.name)
```

### Async Example

```python
async def async_notifier(step_context, state):
    await send_slack_notification(
        f"Step {step_context.step_id} completed"
    )

service.intercept_after_workflow_step(
    fn=async_notifier,
    name="slack-notify"
)
```

---

## Strict vs Non-Strict Mode

### Non-Strict Mode (Default)

Interceptor exceptions are **logged but not propagated**. The workflow continues.

```python
registry = WorkflowInterceptorRegistry(strict=False)
```

**Use case:** Production environments where observability shouldn't break the pipeline.

### Strict Mode

Interceptor exceptions **propagate** and stop the workflow.

```python
registry = WorkflowInterceptorRegistry(strict=True)
```

**Use case:** Testing, development, or when interceptors perform critical validation.

---

## Integration with MemoryService

The `MemoryService` class exposes three methods for interceptor registration.

**Code location:** `src/memu/app/service.py:245-282`

```python
class MemoryService:
    def intercept_before_workflow_step(
        self,
        fn: Callable[..., Any],
        *,
        name: str | None = None,
    ) -> WorkflowInterceptorHandle:
        """Register interceptor to be called before each workflow step."""
        return self._workflow_interceptors.register_before(fn, name=name)

    def intercept_after_workflow_step(
        self,
        fn: Callable[..., Any],
        *,
        name: str | None = None,
    ) -> WorkflowInterceptorHandle:
        """Register interceptor to be called after each workflow step."""
        return self._workflow_interceptors.register_after(fn, name=name)

    def intercept_on_error_workflow_step(
        self,
        fn: Callable[..., Any],
        *,
        name: str | None = None,
    ) -> WorkflowInterceptorHandle:
        """Register interceptor for when a step raises an exception."""
        return self._workflow_interceptors.register_on_error(fn, name=name)
```

---

## Practical Examples

### Example 1: Step Duration Tracking

```python
import time

step_times = {}

def on_step_start(ctx, state):
    step_times[ctx.step_id] = time.time()

def on_step_end(ctx, state):
    duration = time.time() - step_times.get(ctx.step_id, 0)
    print(f"Step {ctx.step_id} took {duration:.2f}s")

service.intercept_before_workflow_step(on_step_start, name="timer-start")
service.intercept_after_workflow_step(on_step_end, name="timer-end")
```

### Example 2: Observability with OpenTelemetry

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def trace_step_start(ctx, state):
    span = tracer.start_span(f"memu.{ctx.workflow_name}.{ctx.step_id}")
    state["_otel_span"] = span

def trace_step_end(ctx, state):
    span = state.get("_otel_span")
    if span:
        span.end()

service.intercept_before_workflow_step(trace_step_start)
service.intercept_after_workflow_step(trace_step_end)
```

### Example 3: Validation Gate

```python
def validate_user_scope(ctx, state):
    if ctx.step_id == "persist_index":
        user = state.get("user")
        if not user or not user.get("user_id"):
            raise ValueError("user_id required for persistence")

service.intercept_before_workflow_step(validate_user_scope, name="scope-check")
```

---

## Execution Flow Functions

The module provides helper functions for running interceptors in the correct order.

**Code location:** `src/memu/workflow/interceptor.py:168-202`

```python
async def run_before_interceptors(
    interceptors: tuple[_WorkflowInterceptor, ...],
    step_context: WorkflowStepContext,
    state: WorkflowState,
    *,
    strict: bool = False,
) -> None:
    """Run all before-step interceptors in registration order."""
    for interceptor in interceptors:
        await _safe_invoke_interceptor(interceptor, strict, step_context, state)

async def run_after_interceptors(...) -> None:
    """Run all after-step interceptors in REVERSE order."""
    for interceptor in reversed(interceptors):
        await _safe_invoke_interceptor(...)

async def run_on_error_interceptors(...) -> None:
    """Run all on-error interceptors in REVERSE order."""
    for interceptor in reversed(interceptors):
        await _safe_invoke_interceptor(..., error)
```

---

## Code References

| Component | File | Line |
|-----------|------|------|
| WorkflowInterceptorRegistry | `src/memu/workflow/interceptor.py` | 56-166 |
| WorkflowStepContext | `src/memu/workflow/interceptor.py` | 16-24 |
| WorkflowInterceptorHandle | `src/memu/workflow/interceptor.py` | 40-53 |
| Service Integration | `src/memu/app/service.py` | 245-282 |
| Execution Functions | `src/memu/workflow/interceptor.py` | 168-218 |

---

## GitHub Discussion Context

- **PR #240** (Workflow step interceptor) - The merged PR that introduced this feature
- This feature enables the event-driven architecture proposed in **PR #218**
- Supports observability requirements from enterprise users

---

*This document is intended for technical copywriters explaining MemU's extensibility model to platform engineers and integration specialists.*
