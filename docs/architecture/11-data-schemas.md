# 11 — Data Schemas & Models

> **In plain English:** This page describes the data structures MiroFlow uses internally — how tasks are logged, how messages are formatted, how tool definitions look. If you're debugging or extending MiroFlow, this is your reference for "what shape does the data have?"

---

## TaskTracer (Pydantic Model)

**File:** `src/logging/task_tracer.py`

Every task execution produces a `TaskTracer` — a structured log that records everything that happened. It's defined with [Pydantic](https://docs.pydantic.dev/) for automatic validation.

```python
class TaskTracer(BaseModel):
    # === IDENTITY ===
    task_id: str                         # Unique identifier for this task
    task_name: str                       # The question/task description
    task_file_name: str | None           # Associated file (if any)
    ground_truth: str | None             # Expected answer (for benchmarks)
    input: Any                           # Raw input data

    # === EXECUTION STATE ===
    status: Literal["pending", "running", "completed", "interrupted", "failed"]
    log_path: Path                       # Where this log is saved on disk

    # === TIMING ===
    start_time: datetime
    end_time: datetime

    # === RESULTS ===
    final_boxed_answer: str              # The extracted \boxed{} answer
    judge_result: str                    # Evaluation result (for benchmarks)
    error: str                           # Error message (if any)

    # === EXECUTION TRACKING ===
    current_main_turn_id: int            # Which turn the main agent is on
    current_sub_agent_turn_id: int       # Which turn the sub-agent is on
    sub_agent_counter: int               # How many sub-agents were spawned
    current_sub_agent_session_id: str | None

    # === CONVERSATION LOGS ===
    main_agent_message_history: dict     # Full main agent conversation
    sub_agent_message_history_sessions: dict  # All sub-agent conversations

    # === STEP-BY-STEP LOG ===
    step_logs: list[StepRecord]          # Detailed execution log
```

Each `StepRecord`:
```python
class StepRecord(BaseModel):
    step_name: str                       # e.g., "tool_call", "llm_response"
    message: str                         # Human-readable description
    timestamp: datetime
    status: Literal["info", "warning", "failed", "success", "debug"]
    metadata: dict[str, Any]             # Extra data (tool name, token count, etc.)
```

The TaskTracer is saved as JSON to disk after every turn and at the end of execution.

> **Design rationale — Pydantic for logging:** Pydantic guarantees the log structure is always valid. You can't accidentally save a TaskTracer with a missing `task_id` or an invalid `status`. This makes logs machine-readable and reliable — important when you're analyzing hundreds of benchmark results programmatically.

---

## BenchmarkTask & BenchmarkResult

**File:** `common_benchmark.py`

Used for batch benchmark evaluation:

```python
@dataclass
class BenchmarkTask:
    task_id: str
    task_question: str
    ground_truth: str
    file_path: Optional[str]
    metadata: Dict[str, Any]
    model_response: str               # The agent's full response
    model_boxed_answer: str           # The extracted \boxed{} answer
    status: TaskStatus                # pending | run_failed | run_completed | result_judged

@dataclass
class BenchmarkResult:
    task_id: str
    task_question: str
    ground_truth: str
    file_path: Optional[str]
    model_response: str
    model_boxed_answer: str
    status: str
    metadata: Dict[str, Any]
```

> **Design rationale — Separate Task vs. Result:** A `BenchmarkTask` represents work to be done (it has a `status` that changes). A `BenchmarkResult` is the final, immutable output. This distinction makes the pipeline clearer: tasks flow in, results flow out.

---

## Message History Format

Messages follow the standard OpenAI/Anthropic convention — a list of objects with `role` and `content`:

```python
# User message (can have multiple parts — text + images)
{
    "role": "user",
    "content": [
        {"type": "text", "text": "What is the capital of France?"},
        {"type": "image_url", "image_url": {"url": "data:image/png;base64,..."}}
    ]
}

# Assistant message (the LLM's response)
{
    "role": "assistant",
    "content": [
        {"type": "text", "text": "I'll search for this information..."}
    ]
}

# Tool result message
{
    "role": "user",    # or "tool" for OpenAI function-call style
    "content": [
        {"type": "text", "text": "Search results: ..."}
    ]
}
```

### Message ID Injection

When `add_message_id: true` is set in the config, user messages get prefixed with a unique ID like `[msg_a1b2c3d4]`. This prevents LLM provider caching from returning responses from different conversations that happen to have similar context.

> **Design rationale — Standard message format:** Using the same format as OpenAI/Anthropic APIs means MiroFlow doesn't need to translate between internal and external formats. Messages flow directly to the API with minimal transformation.

---

## Tool Definition Schema

MCP tool definitions look like this:

```python
[
    {
        "name": "tool-searching",           # Server name
        "tools": [
            {
                "name": "google_search",    # Tool name
                "description": "Search Google for...",
                "schema": {                 # JSON Schema for arguments
                    "type": "object",
                    "properties": {
                        "query": {"type": "string", "description": "..."}
                    },
                    "required": ["query"]
                }
            }
        ]
    }
]
```

For OpenAI function-calling, this is converted to:

```python
{
    "type": "function",
    "function": {
        "name": "tool-searching-google_search",   # server + tool combined
        "description": "Search Google for...",
        "parameters": { ... }                      # Same JSON Schema
    }
}
```

> **Design rationale — MCP-native internally, convert for OpenAI:** MCP is the source of truth for tool definitions. The conversion to OpenAI format happens only when needed, in the provider-specific code. This keeps the core system vendor-neutral.

---

## Tool Result Schema

```python
# Success
{
    "server_name": "tool-searching",
    "tool_name": "google_search",
    "result": "{ ... JSON search results ... }"
}

# Error
{
    "server_name": "tool-searching",
    "tool_name": "google_search",
    "error": "Tool call failed: timeout"
}
```

> **Design rationale — Uniform result format:** Whether a tool succeeds or fails, the result has the same structure with the same keys. This simplifies error handling in the orchestrator — it just checks for the `"error"` key.

---

**Next:** [12 — Logging & Observability](12-logging.md) · **Previous:** [10 — Data Flow](10-data-flow.md)
