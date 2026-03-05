# 10 — Data Flow (End to End)

> **In plain English:** This page traces a single request from the moment you type a command to the moment you get an answer. Follow along to see how data moves through the system.

---

## The Journey of a Question

Let's trace what happens when you run:

```bash
uv run main.py trace \
  --config_file_name=agent_quickstart_reading \
  --task="What is the first country in the XLSX file starting with Co?" \
  --task_file_name="data/FSI-2023-DOWNLOAD.xlsx"
```

### Stage 1: CLI → Config

```
Your command
     │
     ├── task: "What is the first country..."
     ├── task_file_name: "data/FSI-2023-DOWNLOAD.xlsx"
     └── config_file_name: "agent_quickstart_reading"
           │
           ▼
     Hydra resolves the YAML config
     + merges benchmark config
     + resolves ${oc.env:...} variables
```

**What happens:** Python Fire routes your command to `trace_single_task.main()`. Hydra loads `config/agent_quickstart_reading.yaml`, pulls in any referenced benchmark config, and substitutes environment variables (like API keys from `.env`).

### Stage 2: Config → Pipeline Components

```
     Resolved config
           │
           ▼
     create_pipeline_components()
           │
           ├──▶ ToolManager (main agent) — connected to configured MCP servers
           ├──▶ ToolManager (sub-agents) — one per sub-agent
           └──▶ OutputFormatter
```

**What happens:** The pipeline reads the config and creates all the runtime objects — tool managers that know how to connect to MCP servers, and an output formatter.

### Stage 3: Pipeline → Orchestrator

```
     Pipeline components
           │
           ▼
     execute_task_pipeline()
           │
           ├──▶ LLMClient (main agent) — factory creates the right provider
           ├──▶ LLMClient (sub-agent)
           ├──▶ TaskTracer — starts recording
           └──▶ Orchestrator — receives all of the above
```

**What happens:** LLM clients are created (the factory function picks the right provider based on config). A TaskTracer starts recording. The Orchestrator is assembled with all components.

### Stage 4: Orchestrator — The Agent Loop

```
     Orchestrator.run_main_agent()
           │
           ▼
     1. process_input()
        → Detects XLSX file
        → Creates message: "A spreadsheet file is associated..."
           │
           ▼
     2. extract_hints() [if enabled]
        → LLM pre-analyzes: "Be careful with 'starting with Co' — check column headers"
           │
           ▼
     3. get_all_tool_definitions()
        → Connects to MCP servers
        → Gets list of available tools
           │
           ▼
     4. generate_system_prompt()
        → Creates comprehensive instructions for the agent
           │
           ▼
     5. MAIN LOOP:
        ┌────────────────────────────────┐
        │  LLM: "I should read the file" │
        │  → Calls tool: read_file       │
        │  → Gets spreadsheet data       │
        │                                │
        │  LLM: "Looking for Co..."      │
        │  → Finds "Congo Dem. Republic" │
        │  → No more tools needed        │
        └────────────────────────────────┘
           │
           ▼
     6. Generate summary
        → LLM: "The answer is \boxed{Congo Democratic Republic}"
           │
           ▼
     7. extract_final_answer() [if enabled]
        → Validates the boxed answer
           │
           ▼
     8. format_final_summary()
        → Extracts "Congo Democratic Republic" from \boxed{}
```

### Stage 5: Output

```
     Final output
           │
           ├── final_answer: "The first country starting with Co is..."
           ├── boxed_answer: "Congo Democratic Republic"
           └── log_path: "logs/task_xxxx.json"  (full TaskTracer JSON)
```

---

## Complete Diagram

```
User Input (CLI)
    │
    ├── task_description: str
    ├── task_file_name: str (optional)
    └── config_file_name: str
          │
          ▼
    ┌─────────────────┐
    │   Hydra Config  │  ← Resolves YAML + env vars
    │   Resolution    │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────────────────────┐
    │   create_pipeline_components()  │
    │                                 │
    │  ┌─ ToolManager (main agent)    │  ← MCP server configs
    │  ├─ ToolManager (sub-agents)    │
    │  └─ OutputFormatter             │
    └────────────┬────────────────────┘
                 │
                 ▼
    ┌─────────────────────────────────┐
    │   execute_task_pipeline()       │
    │                                 │
    │  ┌─ LLMClient (main)           │  ← Provider factory
    │  ├─ LLMClient (sub)            │
    │  ├─ TaskTracer                  │  ← Pydantic log model
    │  └─ Orchestrator                │
    └────────────┬────────────────────┘
                 │
                 ▼
    ┌─────────────────────────────────┐
    │   Orchestrator.run_main_agent() │
    │                                 │
    │  MAIN LOOP:                     │
    │  ┌─────────────────────┐        │
    │  │ LLM call            │        │  ← message_history → LLM → response
    │  │   ↓                 │        │
    │  │ Parse tool calls    │        │  ← XML or function-call format
    │  │   ↓                 │        │
    │  │ Execute tools       │        │  ← MCP servers or sub-agent loops
    │  │   ↓                 │        │
    │  │ Update msg history  │        │  ← Append tool results
    │  └─────────────────────┘        │
    │                                 │
    │  Generate summary + extract     │
    └────────────┬────────────────────┘
                 │
                 ▼
    ┌─────────────────────────────────┐
    │   Output                        │
    │                                 │
    │  ├── final_answer: str          │
    │  ├── boxed_answer: str          │
    │  └── log_path: Path             │
    └─────────────────────────────────┘
```

> **Design rationale — Linear data flow:** Data flows in one direction (CLI → Config → Pipeline → Orchestrator → Output) with no circular dependencies. This makes the system easy to understand and debug. If something goes wrong, you can trace backwards from the output to find where the problem occurred.

---

**Next:** [11 — Data Schemas](11-data-schemas.md) · **Previous:** [09 — Tool System](09-tool-system.md)
