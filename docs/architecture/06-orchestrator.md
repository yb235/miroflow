# 06 — Orchestrator (The Brain)

> **In plain English:** The orchestrator is the engine that drives the entire agent. It's a loop: ask the LLM what to do → execute the tool it requests → feed the result back → repeat until the LLM has a final answer. This page explains exactly how that loop works.

**File:** `src/core/orchestrator.py`

---

## The Main Agent Loop

```python
async def run_main_agent(self, task_description, task_file_name, task_id):
```

Here's the step-by-step flow:

### Step 1: Input Processing
The orchestrator calls `process_input()` which:
- Detects the file type (PDF? Image? Spreadsheet?) if a file is provided.
- Creates the initial user message with helpful hints like "A PDF file 'report.pdf' is associated with this task."

### Step 2: Hint Generation (Optional)
If enabled in config, a separate LLM call (`extract_hints()`) pre-analyzes the question for tricky aspects before the main agent even starts working.

> **Why?** Some benchmark questions have subtle gotchas (e.g., "the third item" might be ambiguous). Identifying these upfront helps the agent avoid mistakes.

### Step 3: Tool Discovery
The orchestrator connects to all configured MCP servers and fetches their tool definitions (name, description, argument schema). It also adds sub-agents as tools.

### Step 4: System Prompt Generation
The prompt class generates a comprehensive system prompt that includes:
- Instructions for how the agent should behave.
- Descriptions of all available tools.
- Format requirements for tool calls and final answers.

### Step 5: The Main Loop

```
while turn_count < max_turns:
    1. Call the LLM with system prompt + full message history
    2. Parse the response for tool calls
    3. If no tool calls → the agent is done, break out of loop
    4. Execute each tool call:
       - If server name starts with "agent-" → spawn a sub-agent loop
       - Otherwise → call the MCP tool server
    5. Append tool results to message history
    6. Increment turn counter
```

### Step 6: Summary Generation
After the loop ends, the orchestrator asks the LLM to produce a final summary with the answer in `\boxed{}` format.

### Step 7: Final Answer Extraction (Optional)
A separate LLM call validates and cleans up the boxed answer.

### Step 8: Output Formatting
The `\boxed{}` content is extracted, and the final log is formatted and saved.

> **Design rationale — Why a loop with explicit turns?**
> The loop structure gives the agent *iterative refinement*. Rather than trying to answer in one shot, the agent can:
> - Search for initial information.
> - Realize it needs more detail.
> - Search again or read a specific document.
> - Combine everything into a final answer.
>
> The `max_turns` limit is a safety net that prevents infinite loops (and infinite API costs).

---

## The Sub-Agent Loop

```python
async def run_sub_agent(self, sub_agent_name, task_description, keep_tool_result):
```

The sub-agent loop follows the **exact same pattern** as the main agent loop, but with these differences:

| Aspect | Main Agent | Sub-Agent |
|--------|-----------|-----------|
| LLM | Can use an expensive model (e.g., Claude 3.7) | Can use a different (possibly cheaper) model |
| Tools | Usually just reasoning + sub-agent delegation | Has the "hands-on" tools (search, read, code, etc.) |
| Turns | Has its own max_turns limit | Has its own max_turns limit |
| Token tracking | Tracked independently | Tracked independently, reset each session |
| Output | Produces a `\boxed{}` answer | Returns a text summary to the main agent |

> **Design rationale — Same loop, different config:** Reusing the exact same loop structure for both main and sub-agents reduces code duplication and makes the behavior predictable. The only differences are which LLM and tools are available — everything else (parsing, execution, error handling) is identical.

---

## Context-Limit Retry Logic

```python
async def _handle_summary_with_context_limit_retry(self, ...):
```

Sometimes, after many turns of conversation, the message history gets too long and exceeds the LLM's context window. The orchestrator handles this gracefully:

1. Try to generate the summary.
2. If the context window is exceeded:
   - Remove the most recent assistant+user dialogue pair.
   - Mark the task as "failed" (some information was lost).
   - Retry the summary generation.
3. Keep removing dialogue pairs until the summary fits.
4. If only the initial system+user messages remain and it still fails, return an error.

> **Design rationale — Graceful degradation:** Instead of crashing when the context is too long, the system *always* produces some output. Even if it had to drop some conversation turns, the agent still has its initial instructions and the task description. A partial answer is better than no answer — especially during benchmark evaluation where you want results for every task.

---

## Visual Summary

```
User Question
     │
     ▼
┌──────────────────────────┐
│  1. Process Input         │  ← Detect file type, create initial message
│  2. Generate Hints        │  ← [optional] Pre-analyze question
│  3. Discover Tools        │  ← Connect to MCP servers
│  4. Generate System Prompt│  ← Load prompt class
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│  MAIN LOOP               │
│  ┌────────────────────┐  │
│  │ LLM Call           │  │
│  │   ↓                │  │
│  │ Parse Tool Calls   │  │
│  │   ↓                │  │
│  │ Execute Tools      │──│──▶ Sub-agent? → Full sub-agent loop
│  │   ↓                │  │    MCP tool?  → Call tool server
│  │ Update History     │  │
│  └────────────────────┘  │
│  (repeat until done)     │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│  5. Generate Summary      │  ← Request \boxed{} answer
│  6. Extract Final Answer  │  ← [optional] Validate answer
│  7. Format Output         │  ← Extract \boxed{} content
└──────────────────────────┘
```

---

**Next:** [07 — Pipeline Layer](07-pipeline.md) · **Previous:** [05 — Agent Architecture](05-agent-architecture.md)
