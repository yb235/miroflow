# MiroFlow — Deep Architecture & Design Document

> **MiroFlow** is a high-performance, modular research agent framework by MiromindAI.
> It orchestrates LLM-powered agents that use tools (via MCP servers) to solve complex, multi-step reasoning tasks — from future event prediction to benchmark evaluations.

---

## 📖 Beginner-Friendly Version

**New to MiroFlow?** We've broken this document into individual, bite-sized pages with plain-English explanations and design rationale for every decision. Start here:

### → **[docs/architecture/README.md](architecture/README.md)**

| Page | Topic |
|------|-------|
| [01 — Overview](architecture/01-overview.md) | High-level architecture & the seven layers |
| [02 — Directory Structure](architecture/02-directory-structure.md) | Where everything lives |
| [03 — Entry Points & CLI](architecture/03-entry-points-and-cli.md) | CLI commands & usage |
| [04 — Configuration System](architecture/04-configuration-system.md) | Hydra/YAML configuration explained |
| [05 — Agent Architecture](architecture/05-agent-architecture.md) | Main/sub-agent hierarchy & prompts |
| [06 — Orchestrator](architecture/06-orchestrator.md) | The brain: agent loops & retry logic |
| [07 — Pipeline Layer](architecture/07-pipeline.md) | Component assembly |
| [08 — LLM Client](architecture/08-llm-client.md) | LLM provider abstraction |
| [09 — Tool System](architecture/09-tool-system.md) | MCP servers & tool execution |
| [10 — Data Flow](architecture/10-data-flow.md) | End-to-end request tracing |
| [11 — Data Schemas](architecture/11-data-schemas.md) | Data models & structures |
| [12 — Logging](architecture/12-logging.md) | ZMQ logging & observability |
| [13 — Benchmarks](architecture/13-benchmarks.md) | Benchmark evaluation system |
| [14 — Input/Output](architecture/14-input-output.md) | Input/output processing |
| [15 — Design Decisions](architecture/15-design-decisions.md) | All design rationales in one place |
| [16 — API Reference](architecture/16-api-reference.md) | Core classes & functions |
| [17 — Dependencies](architecture/17-dependencies.md) | Library overview |

---

## Full Reference (Original Document)

The rest of this page contains the complete, detailed reference document. For the beginner-friendly version, see the individual pages linked above.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Directory Structure](#2-directory-structure)
3. [Entry Points & CLI](#3-entry-points--cli)
4. [Configuration System](#4-configuration-system)
5. [Agent Architecture](#5-agent-architecture)
   - 5.1 [Main Agent / Sub-Agent Hierarchy](#51-main-agent--sub-agent-hierarchy)
   - 5.2 [Prompt System](#52-prompt-system)
6. [Orchestrator — The Brain](#6-orchestrator--the-brain)
   - 6.1 [Main Agent Loop](#61-main-agent-loop)
   - 6.2 [Sub-Agent Loop](#62-sub-agent-loop)
   - 6.3 [Context-Limit Retry Logic](#63-context-limit-retry-logic)
7. [Pipeline Layer](#7-pipeline-layer)
8. [LLM Client Abstraction](#8-llm-client-abstraction)
   - 8.1 [Factory Pattern](#81-factory-pattern)
   - 8.2 [Provider Client Base Class](#82-provider-client-base-class)
   - 8.3 [Concrete Providers](#83-concrete-providers)
   - 8.4 [Token Usage Tracking](#84-token-usage-tracking)
9. [Tool System — MCP Servers](#9-tool-system--mcp-servers)
   - 9.1 [ToolManager](#91-toolmanager)
   - 9.2 [Tool Configuration](#92-tool-configuration)
   - 9.3 [Available MCP Servers](#93-available-mcp-servers)
   - 9.4 [Tool Execution Flow](#94-tool-execution-flow)
   - 9.5 [Auto-Correction & Error Handling](#95-auto-correction--error-handling)
10. [Data Flow — End to End](#10-data-flow--end-to-end)
11. [Data Schemas & Models](#11-data-schemas--models)
    - 11.1 [TaskTracer (Pydantic)](#111-tasktracer-pydantic)
    - 11.2 [BenchmarkTask & BenchmarkResult](#112-benchmarktask--benchmarkresult)
    - 11.3 [Message History Format](#113-message-history-format)
    - 11.4 [Tool Definition Schema](#114-tool-definition-schema)
    - 11.5 [Tool Result Schema](#115-tool-result-schema)
12. [Logging & Observability](#12-logging--observability)
    - 12.1 [ZMQ-Based Cross-Process Logging](#121-zmq-based-cross-process-logging)
    - 12.2 [Task Context Variables](#122-task-context-variables)
13. [Benchmark Evaluation System](#13-benchmark-evaluation-system)
14. [Input/Output Processing](#14-inputoutput-processing)
15. [Interesting & Noteworthy Design Decisions](#15-interesting--noteworthy-design-decisions)
16. [API Reference Summary](#16-api-reference-summary)
17. [Dependency Overview](#17-dependency-overview)

---

## 1. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                          CLI (fire)                                │
│                    main.py / common_benchmark.py                   │
└────────────────────────┬───────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│               Hydra Configuration System                          │
│        YAML configs (agent, benchmark, tool)                      │
└────────────────────────┬───────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                 Pipeline Layer (pipeline.py)                       │
│     create_pipeline_components() / execute_task_pipeline()         │
└─────────┬────────────────┬────────────────┬───────────────────────┘
          │                │                │
          ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌─────────────────────┐
│  LLM Client  │ │ ToolManager  │ │  OutputFormatter    │
│  (Factory)   │ │  (MCP)       │ │                     │
└──────┬───────┘ └──────┬───────┘ └─────────────────────┘
       │                │
       ▼                ▼
┌────────────────────────────────────────────────────────────────────┐
│                    Orchestrator                                     │
│          run_main_agent() / run_sub_agent()                        │
│                                                                    │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │              Main Agent Loop                              │     │
│   │  ┌──────┐   ┌───────────┐   ┌──────────────────┐        │     │
│   │  │ LLM  │──▶│ Parse XML │──▶│ Execute Tools     │        │     │
│   │  │ Call  │   │ Tool Calls│   │ (MCP / Sub-Agent) │        │     │
│   │  └──────┘   └───────────┘   └────────┬─────────┘        │     │
│   │       ▲                              │                    │     │
│   │       └──────────────────────────────┘                    │     │
│   │              (loop until done or max_turns)               │     │
│   └──────────────────────────────────────────────────────────┘     │
│                                                                    │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │              Sub-Agent Loop (spawned on demand)           │     │
│   │              Same pattern: LLM ↔ Tools                    │     │
│   └──────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                 MCP Tool Servers (FastMCP)                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │
│  │ Searching│ │ Reading  │ │ Reasoning│ │  Vision  │ │ Audio  │  │
│  │ (Serper/ │ │ (Markitd │ │ (OpenAI/ │ │ (Claude/ │ │(Whisper│  │
│  │  Jina)   │ │  own)    │ │ Anthrop.)│ │ Gemini)  │ │  etc.) │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └────────┘  │
│  ┌──────────┐ ┌──────────┐                                        │
│  │  Python  │ │Playwright│                                        │
│  │  (E2B)   │ │(Browser) │                                        │
│  └──────────┘ └──────────┘                                        │
└────────────────────────────────────────────────────────────────────┘
```

MiroFlow follows a layered architecture:

1. **CLI Layer** — `fire`-based commands for single-task tracing and batch benchmark evaluation.
2. **Configuration Layer** — Hydra + YAML for agent, LLM, tool, and benchmark configs.
3. **Pipeline Layer** — Assembles LLM clients, tool managers, and output formatters.
4. **Orchestrator Layer** — The core agentic loop: LLM reasoning → tool execution → feedback → repeat.
5. **LLM Layer** — Provider-agnostic abstraction over OpenAI, Anthropic, DeepSeek, MiroThinker, etc.
6. **Tool Layer** — MCP (Model Context Protocol) servers exposing search, reading, reasoning, vision, audio, code execution, and browser tools.
7. **Logging Layer** — ZMQ-based cross-process log aggregation, Pydantic-based task tracing.

---

## 2. Directory Structure

```
miroflow/
├── main.py                          # CLI entry point (fire)
├── common_benchmark.py              # Benchmark evaluation runner (abstract base)
├── pyproject.toml                   # Project metadata & dependencies
├── .env.template                    # API key template
│
├── config/                          # Hydra configuration
│   ├── __init__.py                  # config_path(), config_name(), debug_config()
│   ├── agent_*.yaml                 # Agent configuration files (per benchmark/model)
│   ├── agent_prompts/               # Python prompt classes
│   │   ├── base_agent_prompt.py     # Abstract base class
│   │   ├── main_agent_prompt_gaia.py
│   │   ├── main_agent_prompt_deepseek.py
│   │   ├── main_boxed_answer.py
│   │   └── sub_worker.py
│   ├── benchmark/                   # Benchmark dataset configs (GAIA, HLE, etc.)
│   └── tool/                        # Tool server configs (YAML per tool)
│
├── src/
│   ├── core/
│   │   ├── orchestrator.py          # Main + Sub agent loops, tool dispatch
│   │   └── pipeline.py              # Pipeline assembly, task execution entry
│   │
│   ├── llm/
│   │   ├── client.py                # LLMClient factory function
│   │   ├── provider_client_base.py  # Abstract base dataclass for all providers
│   │   ├── util.py                  # Timeout decorator
│   │   └── providers/
│   │       ├── claude_openrouter_client.py
│   │       ├── claude_anthropic_client.py
│   │       ├── gpt_openai_client.py
│   │       ├── gpt5_openai_client.py
│   │       ├── deepseek_openrouter_client.py
│   │       └── mirothinker_sglang_client.py
│   │
│   ├── tool/
│   │   ├── manager.py               # ToolManager: connects to MCP servers
│   │   └── mcp_servers/
│   │       ├── searching_mcp_server.py   # Google/Serper search + Jina scraping
│   │       ├── reading_mcp_server.py     # File reading (PDF, XLSX, DOCX, etc.)
│   │       ├── reasoning_mcp_server.py   # Pure text reasoning via LLM
│   │       ├── vision_mcp_server.py      # Image/video analysis
│   │       ├── audio_mcp_server.py       # Audio transcription
│   │       ├── python_server.py          # E2B sandboxed Python execution
│   │       ├── browser_session.py        # Persistent Playwright session
│   │       └── utils/
│   │           ├── smart_request.py      # Smart HTTP with fallbacks
│   │           └── url_unquote.py
│   │
│   ├── utils/
│   │   ├── io_utils.py              # Input processing, OutputFormatter, \boxed{} extraction
│   │   ├── parsing_utils.py         # JSON repair, XML tool-call parsing
│   │   ├── summary_utils.py         # Hint generation, final answer extraction
│   │   └── tool_utils.py            # MCP server param creation, sub-agent exposure
│   │
│   └── logging/
│       ├── logger.py                # bootstrap_logger, ZMQ log handler/listener
│       └── task_tracer.py           # Pydantic TaskTracer model
│
├── utils/                           # Evaluation & utility scripts
│   ├── eval_utils.py
│   ├── eval_answer_from_log.py
│   ├── calculate_average_score.py
│   ├── calculate_score_from_log.py
│   ├── trace_single_task.py
│   └── prepare_benchmark/
│
├── scripts/                         # Shell scripts for batch benchmark runs
├── data/                            # Sample data files
└── docs/                            # Documentation
    └── mkdocs/                      # MkDocs site source
```

---

## 3. Entry Points & CLI

MiroFlow uses [Python Fire](https://github.com/google/python-fire) to expose CLI commands from `main.py`:

| Command | Function | Description |
|---------|----------|-------------|
| `trace` | `utils.trace_single_task.main` | Run a single task with an agent |
| `common-benchmark` | `common_benchmark.main` | Run batch benchmark evaluation |
| `eval-answer` | `utils.eval_answer_from_log.main` | Evaluate answers from logs |
| `avg-score` | `utils.calculate_average_score.main` | Calculate average benchmark scores |
| `score-from-log` | `utils.calculate_score_from_log.main` | Score from existing log files |
| `prepare-benchmark` | `utils.prepare_benchmark.main` | Download and prepare benchmark datasets |
| `print-config` | `print_config` | Debug: print resolved Hydra configuration |

**Example invocation:**

```bash
uv run main.py trace \
  --config_file_name=agent_quickstart_reading \
  --task="What is the first country...?" \
  --task_file_name="data/FSI-2023-DOWNLOAD.xlsx"
```

---

## 4. Configuration System

MiroFlow uses **Hydra** for hierarchical YAML configuration with OmegaConf variable interpolation.

### 4.1 Configuration Hierarchy

```
config/
├── agent_*.yaml           ← Top-level agent configs (select benchmark + override defaults)
│   defaults:
│     - benchmark: gaia-validation   ← Selects benchmark/*.yaml
│     - _self_
│
├── benchmark/*.yaml       ← Dataset-specific settings (name, data paths, eval logic)
└── tool/*.yaml            ← Tool server configs (command, args, env vars)
```

### 4.2 Agent Configuration Structure

Each `agent_*.yaml` defines a complete agent setup:

```yaml
main_agent:
  prompt_class: "MainAgentPrompt_GAIA"        # Python class for prompt generation
  llm:
    provider_class: "ClaudeOpenRouterClient"   # LLM provider implementation
    model_name: "anthropic/claude-3.7-sonnet"
    async_client: true
    temperature: 0.3
    top_p: 0.95
    max_tokens: 32000
    openrouter_api_key: "${oc.env:OPENROUTER_API_KEY}"
    # ... other LLM params
  tool_config:
    - tool-reasoning                            # References config/tool/tool-reasoning.yaml
  max_turns: -1                                 # -1 = unlimited
  max_tool_calls_per_turn: 10
  input_process:
    hint_generation: true                       # Pre-processing: generate hints
  output_process:
    final_answer_extraction: true               # Post-processing: extract final answer
  keep_tool_result: -1                          # -1 = keep all tool results in context
  chinese_context: false                        # Enable Chinese language handling
  add_message_id: true                          # Add unique IDs to user messages

sub_agents:
  agent-worker:                                 # Name MUST start with "agent-"
    prompt_class: SubAgentWorkerPrompt
    llm: { ... }                                # Can use different LLM than main
    tool_config:
      - tool-searching
      - tool-image-video
      - tool-reading
      - tool-code
      - tool-audio
    max_turns: -1
    max_tool_calls_per_turn: 10
```

### 4.3 Tool Configuration Structure

Each tool YAML (`config/tool/tool-*.yaml`) maps to an MCP server:

```yaml
name: "tool-searching"
tool_command: "python"                           # Resolved to sys.executable
args:
  - "-m"
  - "src.tool.mcp_servers.searching_mcp_server"
env:
  SERPER_API_KEY: "${oc.env:SERPER_API_KEY}"
  JINA_API_KEY: "${oc.env:JINA_API_KEY}"
```

### 4.4 Environment Variable Interpolation

Hydra's `${oc.env:VAR_NAME,default}` syntax pulls API keys and settings from `.env`:

```
OPENROUTER_API_KEY=sk-or-v1-...
SERPER_API_KEY=...
JINA_API_KEY=...
E2B_API_KEY=...
OPENAI_API_KEY=...
```

---

## 5. Agent Architecture

### 5.1 Main Agent / Sub-Agent Hierarchy

MiroFlow implements a **hierarchical multi-agent system**:

```
┌─────────────────────────────────────────────────────┐
│                   Main Agent                         │
│  - Has access to reasoning tools                     │
│  - Orchestrates high-level task strategy             │
│  - Can delegate subtasks to sub-agents               │
│  - Generates final summary with \boxed{} answer     │
│                                                      │
│  Tool access: tool-reasoning (+ sub-agent tools)     │
└──────────┬──────────────────────────────────────────┘
           │  Delegates via tool calls to "agent-*"
           ▼
┌─────────────────────────────────────────────────────┐
│              Sub-Agent (agent-worker)                 │
│  - Executes specific information-gathering subtasks  │
│  - Has access to web search, file reading, code,     │
│    vision, audio tools                               │
│  - Returns detailed findings to main agent           │
│                                                      │
│  Tool access: search, read, code, vision, audio      │
└─────────────────────────────────────────────────────┘
```

**Key insight**: Sub-agents are **exposed to the main agent as tools**. When the main agent calls `agent-worker`, the orchestrator spawns a full sub-agent loop (its own LLM ↔ tool cycle) and returns the result as a tool response.

This is implemented in `tool_utils.py::expose_sub_agents_as_tools()` which creates tool definitions from sub-agent prompt classes, and in `orchestrator.py` where `server_name.startswith("agent-")` triggers `run_sub_agent()` instead of MCP tool execution.

### 5.2 Prompt System

Prompts are Python classes inheriting from `BaseAgentPrompt`:

```
BaseAgentPrompt (ABC)
├── MainAgentPrompt_GAIA          # GAIA benchmark prompt
├── MainAgentPromptBoxedAnswer    # General boxed-answer prompt
├── MainAgentPrompt_DeepSeek      # DeepSeek-specific prompt
└── SubAgentWorkerPrompt          # Sub-agent worker prompt
```

Each prompt class implements:

| Method | Purpose |
|--------|---------|
| `generate_system_prompt_with_mcp_tools(mcp_servers, chinese_context)` | Generates the full system prompt including tool definitions in XML format |
| `generate_summarize_prompt(task_description, task_failed, chinese_context)` | Generates the "wrap up and summarize" prompt at the end |
| `expose_agent_as_tool(subagent_name)` | (Sub-agents only) Creates a tool definition so the main agent can call this agent |

**Tool-call format**: MiroFlow uses **XML-style tool calls** (not OpenAI function calling for XML-parse models):

```xml
<use_mcp_tool>
  <server_name>tool-searching</server_name>
  <tool_name>google_search</tool_name>
  <arguments>
  {"query": "future event prediction 2025"}
  </arguments>
</use_mcp_tool>
```

Some providers (OpenAI GPT) use the **native OpenAI function-calling API** instead — the system dynamically converts MCP tool definitions to the OpenAI format via `convert_tool_definition_to_tool_call()`.

---

## 6. Orchestrator — The Brain

**File**: `src/core/orchestrator.py`

The `Orchestrator` class is the central control loop. It manages the conversation between the LLM and tools.

### 6.1 Main Agent Loop

```python
async def run_main_agent(self, task_description, task_file_name, task_id):
```

**Step-by-step flow:**

1. **Input Processing** — `process_input()` detects file types, creates initial user message with file hints and task guidance.
2. **Hint Generation** (optional) — Uses a separate LLM call (`extract_hints()`) to pre-analyze the question for tricky aspects.
3. **Tool Discovery** — Fetches all tool definitions from MCP servers + exposes sub-agents as tools.
4. **System Prompt Generation** — Loads the prompt class, generates system prompt with tool docs.
5. **Main Loop** (`while turn_count < max_turns`):
   - Call LLM with system prompt + message history + tool definitions.
   - Parse response for tool calls.
   - If no tool calls → break (final answer ready).
   - Execute tool calls (or delegate to sub-agents if `server_name.startswith("agent-")`).
   - Append tool results to message history.
   - Repeat.
6. **Summary Generation** — Request final summary with `\boxed{}` answer.
7. **Final Answer Extraction** (optional) — Use separate LLM to extract/validate the boxed answer.
8. **Output Formatting** — Extract `\boxed{}` content, format final log.

### 6.2 Sub-Agent Loop

```python
async def run_sub_agent(self, sub_agent_name, task_description, keep_tool_result):
```

Follows the same pattern as the main agent loop but:
- Uses the sub-agent's own LLM client (can be a different model).
- Has its own tool set (e.g., search + read + code).
- Has its own max_turns limit.
- Resets and independently tracks token usage.
- Returns a text summary to the main agent (not a structured object).

### 6.3 Context-Limit Retry Logic

```python
async def _handle_summary_with_context_limit_retry(self, ...):
```

When generating the final summary, if the context window is exceeded:
1. Remove the most recent assistant-user dialogue pair.
2. Mark the task as failed (information was lost).
3. Retry the summary generation.
4. Repeat until only the initial system+user messages remain.
5. If still failing, return an error message.

This is a **graceful degradation** strategy that ensures the system always produces _some_ output.

---

## 7. Pipeline Layer

**File**: `src/core/pipeline.py`

### `create_pipeline_components(cfg) → (main_tool_manager, sub_tool_managers, output_formatter)`

Assembles all runtime components from Hydra config:
1. Creates `ToolManager` instances for main agent and each sub-agent.
2. Creates `OutputFormatter`.

### `execute_task_pipeline(cfg, task_name, task_id, ...) → (final_answer, boxed_answer, log_path)`

Executes a single task:
1. Creates `LLMClient` instances (main + sub-agent).
2. Creates `TaskTracer` for logging.
3. Constructs `Orchestrator`.
4. Runs `orchestrator.run_main_agent()`.
5. Handles errors and cleanup.

---

## 8. LLM Client Abstraction

### 8.1 Factory Pattern

**File**: `src/llm/client.py`

```python
def LLMClient(task_id, cfg=None, llm_config=None, task_log=None, **kwargs):
```

A **factory function** (not a class) that dynamically imports and instantiates the correct provider:

```python
providers_module = importlib.import_module("src.llm.providers")
ProviderClass = getattr(providers_module, provider_class)  # e.g., "ClaudeOpenRouterClient"
return ProviderClass(task_id=task_id, task_log=task_log, cfg=config)
```

**Security**: Validates that `provider_class` is a valid Python identifier before `importlib`.

### 8.2 Provider Client Base Class

**File**: `src/llm/provider_client_base.py`

`LLMProviderClientBase` is a Python `@dataclass` (not Pydantic) with abstract methods:

| Abstract Method | Purpose |
|----------------|---------|
| `_create_client(config)` | Create the underlying API client (OpenAI, Anthropic, etc.) |
| `_create_message(system_prompt, messages, tools, ...)` | Make the actual API call |
| `process_llm_response(response, history, agent_type)` | Parse response into text + should_break flag |
| `extract_tool_calls_info(response, text)` | Extract tool calls from LLM response |
| `update_message_history(history, tool_results, exceeded)` | Update conversation history with tool results |
| `handle_max_turns_reached_summary_prompt(history, prompt)` | Special handling for summary prompts |

**Shared functionality**:
- `create_message()` — unified entry point that filters history, calls provider, tracks usage.
- `convert_tool_definition_to_tool_call()` — converts MCP tool defs to OpenAI function-call format.
- `_remove_tool_result_from_messages()` — context optimization: remove old tool results while keeping first and last N user messages.
- `get_usage_log()` — formatted cumulative token usage string.

### 8.3 Concrete Providers

| Class | File | API Used |
|-------|------|----------|
| `ClaudeOpenRouterClient` | `claude_openrouter_client.py` | OpenRouter (routes to Anthropic Claude) |
| `ClaudeAnthropicClient` | `claude_anthropic_client.py` | Anthropic API directly |
| `GPTOpenAIClient` | `gpt_openai_client.py` | OpenAI API (GPT-4o, o1, o3, o4-mini, GPT-4.1) |
| `GPT5OpenAIClient` | `gpt5_openai_client.py` | OpenAI API (GPT-5 specific handling) |
| `DeepSeekOpenRouterClient` | `deepseek_openrouter_client.py` | OpenRouter (routes to DeepSeek) |
| `MiroThinkerSGLangClient` | `mirothinker_sglang_client.py` | SGLang server (local open-source model) |

**Noteworthy provider behaviors**:
- `ClaudeOpenRouterClient` uses XML tool-call parsing, has `ContextLimitError` handling, and uses `tiktoken` for token counting.
- `GPTOpenAIClient` detects OpenAI reasoning models (`o1`, `o3`, `o4`, `gpt-5`) and uses `developer` role instead of `system` role.
- All clients use **retry decorators** (`tenacity`) with exponential backoff.
- Async clients use `AsyncOpenAI` while sync clients use `OpenAI`.

### 8.4 Token Usage Tracking

Every `LLMProviderClientBase` tracks cumulative usage per agent session:

```python
total_input_tokens: int
total_input_cached_tokens: int
total_output_tokens: int
total_output_reasoning_tokens: int
```

Usage is reset at the start of each agent session (`reset_usage_stats()`) and logged at the end via `get_usage_log()`.

---

## 9. Tool System — MCP Servers

MiroFlow uses the **Model Context Protocol (MCP)** as its tool abstraction layer. Each tool is an MCP server — a separate process communicating over stdio or SSE.

### 9.1 ToolManager

**File**: `src/tool/manager.py`

```python
class ToolManager:
    def __init__(self, server_configs, tool_blacklist=None)
    async def get_all_tool_definitions() -> list[dict]
    async def execute_tool_call(server_name, tool_name, arguments) -> dict
```

**Key responsibilities**:
- Connects to MCP servers via `stdio_client()` or `sse_client()`.
- Fetches tool definitions (name, description, JSON schema).
- Executes tool calls with a **900-second timeout** (`@with_timeout(900)`).
- Supports **tool blacklisting** — certain (server, tool) pairs can be excluded.
- Maintains a **persistent PlaywrightSession** for browser tools (avoids reconnection overhead).

### 9.2 Tool Configuration

Tools are configured in `config/tool/tool-*.yaml` and referenced by name in agent configs:

| Config File | Server Name | Python Module |
|-------------|-------------|---------------|
| `tool-searching.yaml` | `tool-searching` | `src.tool.mcp_servers.searching_mcp_server` |
| `tool-searching-serper.yaml` | `tool-searching-serper` | `src.tool.mcp_servers.miroapi_serper_mcp_server` |
| `tool-reading.yaml` | `tool-reading` | `src.tool.mcp_servers.reading_mcp_server` |
| `tool-reasoning.yaml` | `tool-reasoning` | `src.tool.mcp_servers.reasoning_mcp_server` |
| `tool-reasoning-os.yaml` | `tool-reasoning-os` | `src.tool.mcp_servers.reasoning_mcp_server_os` |
| `tool-image-video.yaml` | `tool-image-video` | `src.tool.mcp_servers.vision_mcp_server` |
| `tool-image-video-os.yaml` | `tool-image-video-os` | `src.tool.mcp_servers.vision_mcp_server_os` |
| `tool-audio.yaml` | `tool-audio` | `src.tool.mcp_servers.audio_mcp_server` |
| `tool-audio-os.yaml` | `tool-audio-os` | `src.tool.mcp_servers.audio_mcp_server_os` |
| `tool-code.yaml` | `tool-code` | `src.tool.mcp_servers.python_server` |
| `tool-browsing.yaml` | `tool-browsing` | Playwright MCP |
| `tool-markitdown.yaml` | `tool-markitdown` | markitdown-mcp |

### 9.3 Available MCP Servers

#### Searching Server (`searching_mcp_server.py`)
- **Tools**: `google_search`, `google_news_search`, `scrape_website`, `get_wikipedia_article`, `get_calendar`
- **Backends**: Serper API (Google search), Jina AI (web scraping), Wikipedia API
- **Features**: Configurable result filtering (remove snippets, knowledge graph, answer box)

#### Reading Server (`reading_mcp_server.py`)
- **Tools**: `read_file`
- **Supported formats**: PDF, DOCX, XLSX, CSV, PPT, HTML, ZIP, TXT, JSON
- **Features**: URI-based file access (`file:`, `data:`, `http:` schemes), automatic fallback to web scraping

#### Reasoning Server (`reasoning_mcp_server.py`)
- **Tools**: `reasoning`
- **Purpose**: Pure text reasoning, analysis, logical thinking — delegated to a separate LLM (OpenAI o3 or Anthropic Claude)
- **Use cases**: Complex math, puzzles, integrating information, planning

#### Vision Server (`vision_mcp_server.py`)
- **Tools**: `describe_image`, `visual_question_answering`
- **Backends**: Claude (Anthropic), GPT-4o (OpenAI), Gemini (Google) — selected based on available API keys
- **Features**: Multi-provider fallback, support for local files and URLs

#### Audio Server (`audio_mcp_server.py`)
- **Tools**: `transcribe_audio`
- **Backend**: OpenAI Whisper / GPT-4o audio
- **Features**: Multiple audio format support, URL download with retry

#### Python/Code Server (`python_server.py`)
- **Tools**: `execute_python`, `upload_file_to_sandbox`, `download_file_from_sandbox`
- **Backend**: E2B (sandboxed cloud code interpreter)
- **Features**: Pre-installed Python packages, system packages, timeout management, file upload/download

#### Browser Session (`browser_session.py`)
- **Persistent Playwright MCP session** — maintains browser state across tool calls
- **Features**: Supports both stdio and SSE MCP connections

### 9.4 Tool Execution Flow

```
Main Agent LLM Response
        │
        ▼
┌─────────────────────┐
│  Parse Tool Calls   │  (XML parsing or OpenAI function calls)
│  Extract:           │
│  - server_name      │
│  - tool_name        │
│  - arguments        │
│  - call_id          │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ server_name starts  │──YES──▶  run_sub_agent() → Full sub-agent loop
│ with "agent-" ?     │
└─────────┬───────────┘
          │ NO
          ▼
┌─────────────────────┐
│ server_name ==      │──YES──▶  PlaywrightSession.call_tool()
│ "playwright" ?      │          (persistent browser session)
└─────────┬───────────┘
          │ NO
          ▼
┌─────────────────────┐
│ StdioServerParams?  │──YES──▶  stdio_client → ClientSession → call_tool
└─────────┬───────────┘
          │ NO
          ▼
┌─────────────────────┐
│ SSE URL?            │──YES──▶  sse_client → ClientSession → call_tool
└─────────────────────┘
```

### 9.5 Auto-Correction & Error Handling

When a tool call references a **non-existent server name**, the `ToolManager` performs auto-correction:

1. Searches all servers for a tool matching the `tool_name`.
2. If found in **exactly one server** → auto-corrects and executes, adding a note to the result.
3. If found in **multiple servers** → returns an error listing the options.
4. If not found → returns an error suggesting the names may be confused.

Additionally:
- **HuggingFace scraping protection**: Blocks scraping of HuggingFace datasets/spaces (prevents leaking benchmark answers).
- **Tool result truncation**: Results are truncated to 100,000 characters (≈25k tokens).

---

## 10. Data Flow — End to End

Here is the complete data flow for a single task execution:

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
    │  1. process_input()             │  ← Detect file type, create user msg
    │  2. extract_hints() [optional]  │  ← Pre-analyze question via LLM
    │  3. get_all_tool_definitions()  │  ← Connect to MCP servers
    │  4. generate_system_prompt()    │  ← Load prompt class
    │                                 │
    │  5. MAIN LOOP:                  │
    │     ┌─────────────────────┐     │
    │     │ LLM call            │     │  ← message_history → LLM → response
    │     │   ↓                 │     │
    │     │ Parse tool calls    │     │  ← XML or function-call format
    │     │   ↓                 │     │
    │     │ Execute tools       │     │  ← MCP servers or sub-agent loops
    │     │   ↓                 │     │
    │     │ Update msg history  │     │  ← Append tool results
    │     └─────────────────────┘     │
    │                                 │
    │  6. Generate summary prompt     │  ← Request \boxed{} answer
    │  7. extract_final_answer()      │  ← [optional] LLM validates answer
    │  8. format_final_summary()      │  ← Extract \boxed{} content
    └────────────┬────────────────────┘
                 │
                 ▼
    ┌─────────────────────────────────┐
    │   Output                        │
    │                                 │
    │  ├── final_answer: str          │  ← Full LLM summary text
    │  ├── boxed_answer: str          │  ← Extracted \boxed{} content
    │  └── log_path: Path             │  ← TaskTracer JSON file
    └─────────────────────────────────┘
```

---

## 11. Data Schemas & Models

### 11.1 TaskTracer (Pydantic)

**File**: `src/logging/task_tracer.py`

The primary structured log for every task execution:

```python
class TaskTracer(BaseModel):
    # Identity
    task_id: str
    task_name: str
    task_file_name: str | None
    ground_truth: str | None
    input: Any

    # Execution state
    status: Literal["pending", "running", "completed", "interrupted", "failed"]
    log_path: Path

    # Timing
    start_time: datetime
    end_time: datetime

    # Results
    final_boxed_answer: str
    judge_result: str
    error: str

    # Execution tracking
    current_main_turn_id: int
    current_sub_agent_turn_id: int
    sub_agent_counter: int
    current_sub_agent_session_id: str | None

    # Message histories (full conversation logs)
    main_agent_message_history: dict[str, Any]
    sub_agent_message_history_sessions: dict[str, dict[str, Any]]

    # Step-by-step execution log
    step_logs: list[StepRecord]
```

Each `StepRecord`:
```python
class StepRecord(BaseModel):
    step_name: str
    message: str
    timestamp: datetime
    status: Literal["info", "warning", "failed", "success", "debug"]
    metadata: dict[str, Any]
```

The TaskTracer is saved as JSON to disk after every turn and at the end of execution.

### 11.2 BenchmarkTask & BenchmarkResult

**File**: `common_benchmark.py`

```python
@dataclass
class BenchmarkTask:
    task_id: str
    task_question: str
    ground_truth: str
    file_path: Optional[str]
    metadata: Dict[str, Any]
    model_response: str
    model_boxed_answer: str
    status: TaskStatus  # pending | run_failed | run_completed | result_judged

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

### 11.3 Message History Format

Messages follow the OpenAI/Anthropic convention with `role` + `content`:

```python
# User message (with multi-part content)
{
    "role": "user",
    "content": [
        {"type": "text", "text": "What is the capital of France?"},
        {"type": "image_url", "image_url": {"url": "data:image/png;base64,..."}}
    ]
}

# Assistant message
{
    "role": "assistant",
    "content": [
        {"type": "text", "text": "I'll search for this information..."}
    ]
}

# Tool result message (format varies by provider)
{
    "role": "user",  # or "tool" for OpenAI function-call style
    "content": [
        {"type": "text", "text": "Search results: ..."}
    ]
}
```

**Message ID injection**: When `add_message_id: true`, user messages get prefixed with `[msg_xxxxxxxx]` to prevent cross-conversation cache hits.

### 11.4 Tool Definition Schema

MCP tool definitions follow this structure:

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
        "name": "tool-searching-google_search",   # server-tool combined
        "description": "Search Google for...",
        "parameters": { ... }                      # Same JSON Schema
    }
}
```

### 11.5 Tool Result Schema

Tool execution returns a dictionary:

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

---

## 12. Logging & Observability

### 12.1 ZMQ-Based Cross-Process Logging

**File**: `src/logging/logger.py`

MiroFlow uses **ZeroMQ (ZMQ)** for cross-process log aggregation. This is needed because MCP tool servers run as **separate processes**:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  MCP Server  │    │  MCP Server  │    │  MCP Server  │
│  (searching) │    │  (reading)   │    │  (python)    │
│              │    │              │    │              │
│  ZMQHandler  │    │  ZMQHandler  │    │  ZMQHandler  │
│  (PUSH)      │    │  (PUSH)      │    │  (PUSH)      │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                           ▼
               ┌───────────────────────┐
               │   ZMQ Listener (PULL) │
               │   (main process)      │
               │                       │
               │   Forwards to root    │
               │   Python logger       │
               └───────────────────────┘
```

**Protocol**: `{task_id}||{tool_name}||{message}` over ZMQ PUSH/PULL sockets.

**Port management**: Automatically finds an available port starting from 6000, with fallback to random port binding.

### 12.2 Task Context Variables

`TASK_CONTEXT_VAR` is a Python `ContextVar` that stores the current task ID. This enables:
- Per-task log files via `TaskFilter`.
- Task ID injection into log records via a custom `LogRecordFactory`.
- Task ID propagation to MCP server processes via environment variables.

---

## 13. Benchmark Evaluation System

**File**: `common_benchmark.py`

The benchmark system is built on an abstract `BenchmarkRunner` pattern (implemented as `common_benchmark.main()`):

1. **Load benchmark config** — selects dataset, evaluation criteria.
2. **Load tasks** — from HuggingFace datasets, local files, or custom loaders.
3. **Run tasks concurrently** — uses `asyncio` with configurable concurrency.
4. **Evaluate results** — compares model answers to ground truth using dataset-specific evaluation functions.
5. **Save results** — JSON logs, score summaries, per-task traces.

**Supported benchmarks**:

| Benchmark | Config | Description |
|-----------|--------|-------------|
| GAIA Validation | `gaia-validation.yaml` | General AI assistant benchmark |
| GAIA Test | `gaia-test.yaml` | GAIA test split |
| HLE | `hle.yaml` | Human-Level Evaluation (with/without text-only) |
| BrowseComp EN/ZH | `browsecomp-en.yaml`, `browsecomp-zh.yaml` | Web browsing comprehension |
| FutureX | `futurex.yaml` | Future event prediction |
| xBench DeepSearch | `xbench-ds.yaml` | Cross-lingual deep search |
| FinSearchComp | `finsearchcomp.yaml` | Financial search comprehension |
| WebWalkerQA | `webwalkerqa.yaml` | Web navigation QA |

---

## 14. Input/Output Processing

### Input Processing (`io_utils.py::process_input`)

Detects file types and adds contextual hints:
- Image files (jpg, png, gif, webp)
- Documents (pdf, docx, pptx)
- Spreadsheets (xlsx, xls)
- Audio (wav, mp3, m4a)
- Archives (zip)
- Text (txt), JSON, HTML

Adds to the task description: `"A {file_type} file '{filename}' is associated with this task. Local path: {absolute_path}"`

### Output Processing (`io_utils.py::OutputFormatter`)

**`\boxed{}` extraction**: Uses balanced-brace counting (not regex) to correctly handle nested braces:

```python
def _extract_boxed_content(self, text: str) -> str:
    # Finds the LAST \boxed{...} match in the text
    # Handles arbitrary nesting: \boxed{a + {b + c}}
```

**Tool result formatting**: Truncates to 100k chars, provides error context.

### Hint Generation (`summary_utils.py::extract_hints`)

Pre-analyzes the task question using a separate LLM call to identify:
- Potential pitfalls and tricky aspects
- Multiple valid interpretations
- Formatting requirements
- Potential mistakes in the question itself

### Final Answer Extraction (`summary_utils.py::extract_gaia_final_answer`)

Post-processes the LLM's summary to extract a clean, formatted answer.

---

## 15. Interesting & Noteworthy Design Decisions

### 🔍 Sub-Agents as Tools
Sub-agents are indistinguishable from tools from the main agent's perspective. The main agent calls `agent-worker` with a subtask string, and the orchestrator transparently runs a full agent loop. This enables **recursive agent composition**.

### 🔄 XML Tool Calling for Anthropic Models
Rather than using OpenAI's function-calling API (which Anthropic also supports), MiroFlow uses **XML-formatted tool calls** parsed with regex. This gives full control over the tool-calling format and works uniformly across providers.

### 📨 Message ID Injection
Unique message IDs (`[msg_xxxxxxxx]`) are prepended to user messages to prevent LLM provider caching from returning responses from different conversations with similar context.

### 🧠 Hint Generation as Pre-Processing
Before the main agent starts working, a separate LLM call analyzes the question for subtle pitfalls. This "thinking before doing" approach is used in benchmarks like GAIA where questions may contain tricky or ambiguous wording.

### 🔒 HuggingFace Scraping Protection
The system specifically detects and blocks attempts to scrape HuggingFace dataset pages, preventing the agent from "cheating" on benchmarks by directly reading ground truth data.

### 🌐 Cross-Process Logging with ZMQ
Since MCP tool servers are separate processes (spawned via `stdio_client`), regular Python logging doesn't aggregate their output. ZMQ provides a lightweight pub/sub mechanism to funnel all logs back to the main process.

### 📊 Balanced-Brace `\boxed{}` Extraction
Instead of naive regex, the system uses a character-by-character brace-counting algorithm that correctly handles nested braces like `\boxed{f(x) = \frac{a}{b+c}}`.

### 🔄 Context Window Management
The `keep_tool_result` parameter and `_remove_tool_result_from_messages()` method implement a sliding-window approach to manage context length. Old tool results can be replaced with `"Tool result is omitted to save tokens."` while keeping the first and last N messages.

### 🛡️ Auto-Correction of Tool Server Names
When the LLM hallucinates a server name, the `ToolManager` searches all servers for the requested tool and auto-corrects if there's an unambiguous match.

### 🔄 Provider-Specific Role Handling
OpenAI's newer models (o1, o3, GPT-5) use `developer` role instead of `system`, while older models and Anthropic use `system`. The provider clients handle this transparently.

### 🏗️ E2B Sandboxed Execution
Python code execution uses E2B (a cloud sandbox), not local execution. This provides security isolation and comes pre-installed with dozens of data science packages.

---

## 16. API Reference Summary

### Core Classes

| Class/Function | Location | Description |
|---------------|----------|-------------|
| `Orchestrator` | `src/core/orchestrator.py` | Main + sub-agent loop management |
| `execute_task_pipeline()` | `src/core/pipeline.py` | Single-task execution entry point |
| `create_pipeline_components()` | `src/core/pipeline.py` | Component assembly from config |
| `LLMClient()` | `src/llm/client.py` | Factory function for LLM providers |
| `LLMProviderClientBase` | `src/llm/provider_client_base.py` | Abstract base for all LLM providers |
| `ToolManager` | `src/tool/manager.py` | MCP server connection & tool execution |
| `TaskTracer` | `src/logging/task_tracer.py` | Pydantic model for structured task logs |
| `OutputFormatter` | `src/utils/io_utils.py` | Tool result + final answer formatting |
| `BaseAgentPrompt` | `config/agent_prompts/base_agent_prompt.py` | Abstract base for prompt classes |

### Key Functions

| Function | Location | Description |
|----------|----------|-------------|
| `process_input()` | `src/utils/io_utils.py` | File detection + initial message creation |
| `create_mcp_server_parameters()` | `src/utils/tool_utils.py` | YAML → StdioServerParameters |
| `expose_sub_agents_as_tools()` | `src/utils/tool_utils.py` | Sub-agents → tool definitions |
| `extract_hints()` | `src/utils/summary_utils.py` | Pre-analysis hint generation |
| `extract_gaia_final_answer()` | `src/utils/summary_utils.py` | Post-processing answer extraction |
| `bootstrap_logger()` | `src/logging/logger.py` | Logger setup with Rich console |
| `setup_mcp_logging()` | `src/logging/logger.py` | ZMQ log handler for MCP processes |

---

## 17. Dependency Overview

| Package | Purpose |
|---------|---------|
| `fire` | CLI command dispatch |
| `hydra-core` / `omegaconf` | Configuration management |
| `openai` | OpenAI API client (also used for OpenRouter) |
| `anthropic` | Anthropic Claude API client |
| `google-genai` | Google Gemini API client |
| `mcp` / `fastmcp` | Model Context Protocol server/client |
| `e2b-code-interpreter` | Sandboxed Python execution |
| `tenacity` | Retry logic with exponential backoff |
| `tiktoken` | Token counting for context management |
| `pydantic` | Data validation (TaskTracer) |
| `rich` | Console output formatting |
| `pyzmq` | Cross-process log aggregation |
| `aiohttp` | Async HTTP for tool servers |
| `wikipedia` | Wikipedia API client |
| `json5` | Lenient JSON parsing |
| `markitdown-mcp` | Document-to-markdown conversion |
| `mutagen` | Audio file metadata |
| `Pillow` | Image processing |
| `datasets` | HuggingFace datasets loader |
| `pandas` / `openpyxl` | Data manipulation & Excel support |

---

*Document generated for the MiroFlow repository. For the latest information, refer to the [official documentation](https://miromindai.github.io/MiroFlow/) and the [README](../README.md).*
