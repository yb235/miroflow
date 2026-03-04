# 07 — Pipeline Layer

> **In plain English:** The pipeline layer is the "assembly line" that puts all the pieces together before the orchestrator can run. It reads the config, creates the LLM clients, connects to tool servers, and hands everything to the orchestrator.

**File:** `src/core/pipeline.py`

---

## Two Key Functions

### 1. `create_pipeline_components(cfg)`

**What it does:** Takes the Hydra config and creates all runtime objects.

**Returns:**
- `main_tool_manager` — A `ToolManager` connected to the main agent's MCP servers.
- `sub_tool_managers` — A dictionary of `ToolManager` instances, one per sub-agent.
- `output_formatter` — An `OutputFormatter` for processing results.

**Think of it as:** The factory that builds all the parts before the assembly starts.

### 2. `execute_task_pipeline(cfg, task_name, task_id, ...)`

**What it does:** Executes a single task from start to finish.

**Steps:**
1. Creates `LLMClient` instances for the main agent and each sub-agent (using the factory pattern).
2. Creates a `TaskTracer` for structured logging.
3. Constructs the `Orchestrator` with all components.
4. Calls `orchestrator.run_main_agent()`.
5. Handles errors and cleanup.

**Returns:** `(final_answer, boxed_answer, log_path)`

**Think of it as:** Pressing the "Go" button — it runs one complete task and gives you the result.

---

## Why a Separate Pipeline Layer?

> **Design rationale:**
> 1. **Separation from orchestration:** The orchestrator shouldn't know *how* to create LLM clients or connect to tool servers — it should just *use* them. The pipeline handles the "how," the orchestrator handles the "what."
> 2. **Testability:** You can test the orchestrator by giving it mock LLM clients and mock tool managers, without needing real API connections.
> 3. **Reusability:** Both single-task runs (`trace`) and batch benchmark runs (`common-benchmark`) use the same pipeline logic. The pipeline is the shared entry point.
> 4. **Error isolation:** If something goes wrong during setup (bad API key, missing tool config), it fails in the pipeline layer with a clear error — before the orchestrator even starts.

---

## How It All Fits Together

```
Config (YAML)
     │
     ▼
create_pipeline_components(cfg)
     │
     ├──▶ ToolManager (main agent)
     ├──▶ ToolManager (sub-agent "agent-worker")
     └──▶ OutputFormatter
           │
           ▼
execute_task_pipeline(cfg, task, ...)
     │
     ├──▶ LLMClient (main agent)     ← Factory creates the right provider
     ├──▶ LLMClient (sub-agent)      ← Possibly a different model/provider
     ├──▶ TaskTracer                  ← Structured logging
     └──▶ Orchestrator               ← Gets all the above
           │
           ▼
     orchestrator.run_main_agent()
           │
           ▼
     (final_answer, boxed_answer, log_path)
```

---

**Next:** [08 — LLM Client](08-llm-client.md) · **Previous:** [06 — Orchestrator](06-orchestrator.md)
