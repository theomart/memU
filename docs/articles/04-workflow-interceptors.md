🔬 Real use case where AI agent frameworks bring real value: observability without touching core code.

MemU's workflow interceptors let you hook into EVERY step of the memory pipeline. Here's how.

---

👞 Non technical TLDR

- Think of it like airport security checkpoints: you can add checks before boarding (validation), after landing (logging), or when something goes wrong (alerts)
- MemU lets you add these "checkpoints" to every step of memory processing
- The best part: you don't modify any MemU code, you just register your hooks

---

🔬 Technical TLDR

- **3 types of interceptors**:
  - `before` → runs before each workflow step (validation, auth, timing start)
  - `after` → runs after step completes (logging, metrics, notifications)
  - `on_error` → runs when step fails (alerts, recovery, cleanup)

- **Registration is dead simple**:
```python
handle = service.intercept_before_workflow_step(
    fn=my_validator,
    name="scope-check"
)
# Later: handle.dispose() to remove
```

- **WorkflowStepContext** gives you everything:
  - `workflow_name`: "memorize", "retrieve_rag", "retrieve_llm"
  - `step_id`: "extract_items", "persist_index", etc.
  - `step_role`: "ingest", "extract", "persist"
  - `step_context`: step-specific metadata

- **Async support built-in**: sync or async handlers, automatically awaited

---

🛠 Practical examples

**1. Duration tracking:**
```python
def on_start(ctx, state):
    state["_timer"] = time.time()

def on_end(ctx, state):
    duration = time.time() - state["_timer"]
    print(f"{ctx.step_id}: {duration:.2f}s")
```

**2. OpenTelemetry tracing:**
```python
def trace_start(ctx, state):
    span = tracer.start_span(f"memu.{ctx.step_id}")
    state["_span"] = span

def trace_end(ctx, state):
    state["_span"].end()
```

**3. Validation gate:**
```python
def validate_scope(ctx, state):
    if ctx.step_id == "persist_index":
        if not state.get("user_id"):
            raise ValueError("user_id required")
```

---

🥊 Strict vs Non-Strict Mode

- **Non-strict (default)**: interceptor errors are logged, workflow continues
- **Strict**: interceptor errors propagate, workflow stops

Production = non-strict (observability shouldn't break the pipeline)
Testing = strict (catch bugs early)

---

🍓 Key insight

After-interceptors run in REVERSE order (LIFO). Why?

Think of it like try/finally blocks. If you start a timer in `before`, you want to stop it in `after` - even if other interceptors were registered after yours.

`register(A) → register(B) → before(A) → before(B) → step → after(B) → after(A)`

Clean stack unwinding.

---

This pattern is becoming standard in agent frameworks. Are you using interceptors/hooks in your pipelines?

#ai #llm #observability #agents
