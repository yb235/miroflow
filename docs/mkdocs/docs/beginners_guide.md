# 🧭 MiroFlow: Complete Beginner's Guide

> **Who is this for?** This guide is written for someone who has never used MiroFlow before and wants to understand *everything* — what the project is, how the code is structured, how all the pieces fit together, and how to get things working from scratch.

---

## Table of Contents

1. [What is MiroFlow?](#1-what-is-miroflow)
2. [Repository Layout — Every File Explained](#2-repository-layout-every-file-explained)
3. [Core Architecture](#3-core-architecture)
4. [Step-by-Step Workflow](#4-step-by-step-workflow)
5. [The Configuration System (YAML + .env)](#5-the-configuration-system-yaml-env)
6. [LLM Clients — How MiroFlow Talks to AI Models](#6-llm-clients-how-miroflow-talks-to-ai-models)
7. [Tools — What the Agent Can Do](#7-tools-what-the-agent-can-do)
8. [External APIs You Need](#8-external-apis-you-need)
9. [Command Reference](#9-command-reference)
10. [Running Your First Task (Hands-on)](#10-running-your-first-task-hands-on)
11. [Understanding the Logs](#11-understanding-the-logs)
12. [Running Benchmarks](#12-running-benchmarks)
13. [How to Extend MiroFlow](#13-how-to-extend-miroflow)
14. [Glossary](#14-glossary)

---

## 1. What is MiroFlow?

MiroFlow is an **AI agent framework**. At its heart it does one thing: it takes a task described in plain English, gives an AI model (like Claude or GPT) a set of *tools* it can use, and lets the model solve the task step by step.

Think of it like hiring a smart assistant and equipping them with a laptop, a browser, a calculator, and a library card. The assistant decides which tool to pick up at each step until they have the answer.

**What makes MiroFlow special?**

| Feature | What it means for you |
|---|---|
| **Multi-agent** | A *main agent* coordinates one or more *sub-agents*, each specialised for a different kind of work |
| **Tool ecosystem** | Out of the box: web search, file reading, Python code execution, image/video analysis, audio transcription, advanced reasoning |
| **Any LLM** | Swap between Claude, GPT-4o, GPT-5, DeepSeek, or the open-source MiroThinker with a one-line config change |
| **Benchmark suite** | Run well-known AI benchmarks (GAIA, HLE, BrowseComp, …) and reproduce published results |
| **Observable** | Every agent step, tool call, and LLM message is saved in structured `.log` files |

---

## 2. Repository Layout — Every File Explained

```
miroflow/
│
├── main.py                   # ← START HERE. CLI entry point.
├── pyproject.toml            # Python project metadata and dependencies
├── uv.lock                   # Exact dependency versions (managed by `uv`)
├── .env.template             # Template for API keys — copy to .env and fill in
├── common_benchmark.py       # Logic for running benchmark suites
│
├── config/                   # ─── All YAML configuration lives here ───
│   ├── __init__.py           # Exports helper functions: config_path(), config_name()
│   ├── agent_*.yaml          # One file per agent preset (quickstart, GAIA, HLE, …)
│   ├── agent_prompts/        # Python classes that generate system/user prompts
│   │   ├── base_agent_prompt.py     # Abstract base class every prompt must inherit
│   │   ├── main_boxed_answer.py     # Default main-agent prompt (expects \boxed{} answer)
│   │   ├── main_agent_prompt_gaia.py # GAIA-optimised main-agent prompt
│   │   ├── sub_worker.py            # Default sub-agent prompt
│   │   └── ...
│   ├── benchmark/            # Benchmark-specific settings (dataset paths, concurrency, …)
│   └── tool/                 # One YAML per tool — declares the MCP server command
│       ├── tool-reading.yaml
│       ├── tool-searching.yaml
│       ├── tool-code.yaml
│       └── ...
│
├── src/                      # ─── Core Python library ───
│   ├── core/
│   │   ├── orchestrator.py   # Main agent loop + sub-agent delegation
│   │   └── pipeline.py       # Wires components together for a single task
│   ├── llm/
│   │   ├── client.py         # Factory function: creates the right LLM provider object
│   │   ├── provider_client_base.py  # Abstract base class all LLM clients inherit
│   │   ├── util.py           # Token counting, message trimming helpers
│   │   └── providers/        # Concrete LLM implementations
│   │       ├── claude_openrouter_client.py   # Claude via OpenRouter
│   │       ├── claude_anthropic_client.py    # Claude via Anthropic's own API
│   │       ├── gpt_openai_client.py          # GPT-4o via OpenAI
│   │       ├── gpt5_openai_client.py         # GPT-5 via OpenAI
│   │       ├── deepseek_openrouter_client.py # DeepSeek via OpenRouter
│   │       └── mirothinker_sglang_client.py  # Self-hosted MiroThinker via SGLang
│   ├── tool/
│   │   ├── manager.py        # ToolManager — connects to MCP servers, executes tool calls
│   │   └── mcp_servers/      # One Python module per tool (run as sub-processes)
│   │       ├── searching_mcp_server.py  # Google search + web scraping
│   │       ├── reading_mcp_server.py    # File reading (PDF, XLSX, DOCX, …)
│   │       ├── python_server.py         # Python code execution via E2B sandbox
│   │       ├── reasoning_mcp_server.py  # "Think harder" reasoning via a strong LLM
│   │       ├── vision_mcp_server.py     # Image and video analysis
│   │       ├── audio_mcp_server.py      # Audio transcription
│   │       └── ...
│   ├── logging/
│   │   ├── logger.py         # Structured logger using Rich; TASK_CONTEXT_VAR for async safety
│   │   └── task_tracer.py    # Records every step of a task run to a .log file
│   └── utils/
│       ├── io_utils.py       # OutputFormatter, process_input (attaches files to messages)
│       ├── tool_utils.py     # Helpers: parse tool configs, expose sub-agents as tools
│       └── summary_utils.py  # Extract final boxed answers, hints from LLM text
│
├── utils/                    # ─── CLI utility scripts ───
│   ├── trace_single_task.py  # `uv run main.py trace` — run one task
│   ├── common_benchmark.py   # `uv run main.py common-benchmark` — run a full benchmark
│   ├── eval_answer_from_log.py      # Score answers stored in log files
│   ├── calculate_average_score.py   # Aggregate scores across a run
│   ├── calculate_score_from_log.py  # Alternative scoring from logs
│   └── prepare_benchmark/    # Download and pre-process benchmark datasets
│
├── data/                     # Benchmark datasets (downloaded separately)
├── scripts/                  # Miscellaneous helper scripts
└── docs/                     # Documentation (you are here)
    └── mkdocs/               # MkDocs site source
```

---

## 3. Core Architecture

```
┌─────────────────────────────────────────────────────┐
│                     main.py (CLI)                   │
│  fire.Fire  →  utils/trace_single_task.py::main()  │
└─────────────────────────────┬───────────────────────┘
                              │ hydra loads YAML config
                              ▼
┌─────────────────────────────────────────────────────┐
│              src/core/pipeline.py                   │
│  create_pipeline_components()                       │
│    • ToolManager (main agent)                       │
│    • ToolManager (each sub-agent)                   │
│    • OutputFormatter                                │
│                                                     │
│  execute_task_pipeline()                            │
│    • LLMClient (main agent)                         │
│    • LLMClient (sub-agents)                         │
│    • TaskTracer (structured log)                    │
│    • Orchestrator.run_main_agent()   ───────────────┼──┐
└─────────────────────────────────────────────────────┘  │
                                                          │
┌─────────────────────────────────────────────────────┐  │
│           src/core/orchestrator.py                  │◄─┘
│                                                     │
│  Main-agent loop (one turn = one LLM call):         │
│    1. Build system prompt from prompt_class         │
│    2. Call LLM  →  LLMClient.create_message()       │
│    3. Parse tool calls from response                │
│    4. Execute each tool call via ToolManager        │
│    5. Append tool result to message history         │
│    6. Repeat until LLM says "done"                  │
│                                                     │
│  Sub-agent delegation:                              │
│    • Main agent calls a sub-agent as if it were a  │
│      tool (run_sub_agent())                         │
│    • Sub-agent runs its own inner loop with its     │
│      own tools                                      │
│    • Sub-agent returns a text summary               │
└──────────┬──────────────────────────────────────────┘
           │
     ┌─────▼──────┐    ┌──────────────────────────┐
     │ LLMClient  │    │      ToolManager          │
     │ (provider) │    │                           │
     │            │    │  MCP stdio sub-processes: │
     │ OpenRouter │    │  • tool-searching         │
     │ Anthropic  │    │  • tool-reading           │
     │ OpenAI     │    │  • tool-code              │
     │ SGLang     │    │  • tool-reasoning         │
     └────────────┘    │  • tool-image-video       │
                       │  • tool-audio             │
                       └──────────────────────────┘
```

### 3.1 Main Agent vs Sub-Agents

| | Main Agent | Sub-Agent |
|---|---|---|
| **Role** | High-level planner; decides *what* to do | Executes a specific subtask end-to-end |
| **Tools** | Usually just `tool-reasoning` (thinks harder) plus the ability to call sub-agents | Full tool suite (search, code, files, …) |
| **Config key** | `main_agent:` | `sub_agents.agent-worker:` |
| **Prompt class** | e.g. `MainAgentPromptBoxedAnswer` | e.g. `SubAgentWorkerPrompt` |

If you look at `config/agent_quickstart_reading.yaml`, `sub_agents: null` — this means the main agent *is* the only agent and handles everything itself.

### 3.2 ToolManager and MCP

MiroFlow uses the **Model Context Protocol (MCP)** to talk to tools. Each tool is a tiny Python server that runs as a separate process. The `ToolManager`:

1. Reads the tool config (e.g. `config/tool/tool-searching.yaml`)
2. Launches the MCP server as a child process via `stdio_client`
3. Queries the server for its tool list (`session.list_tools()`)
4. Executes tool calls on demand (`session.call_tool()`)

This design means tools are completely isolated — a crash in a tool never crashes the agent.

---

## 4. Step-by-Step Workflow

Below is a complete trace of what happens when you run:

```bash
uv run main.py trace \
  --config_file_name=agent_quickstart_reading \
  --task="What is the first country starting with Co in the FSI spreadsheet?" \
  --task_file_name="data/FSI-2023-DOWNLOAD.xlsx"
```

### Step 1 — CLI parsing (`main.py`)

`fire.Fire` maps the `trace` command to `utils/trace_single_task.py::main()`.  
`dotenv.load_dotenv()` reads your `.env` file so all `${oc.env:OPENROUTER_API_KEY}` references in YAML resolve correctly.

### Step 2 — Hydra loads the config

```python
with hydra.initialize_config_dir(config_dir=config_path(), version_base=None):
    cfg = hydra.compose(config_name="agent_quickstart_reading", overrides=[])
```

Hydra merges `config/agent_quickstart_reading.yaml` with the referenced `config/benchmark/example_dataset.yaml` and `config/tool/tool-reading.yaml` into a single `DictConfig` object.

### Step 3 — Pipeline components are created (`pipeline.py`)

`create_pipeline_components(cfg)` does three things:

1. **ToolManager** — reads the `tool_config` list from the YAML (e.g. `[tool-reading]`), calls `create_mcp_server_parameters()` to turn each tool name into a `StdioServerParameters` object, and constructs a `ToolManager`.
2. **Sub-agent ToolManagers** — same process for every sub-agent (none in this case).
3. **OutputFormatter** — a simple helper that formats the agent's final answer for display.

### Step 4 — Pipeline executes (`pipeline.py::execute_task_pipeline`)

1. Creates `TaskTracer` — opens the log file, records task metadata.
2. Creates `LLMClient` — instantiates `ClaudeOpenRouterClient` with your API key.
3. Creates `Orchestrator` — passes in the ToolManager, LLMClient, and TaskTracer.
4. Calls `orchestrator.run_main_agent(task_description, task_file_name)`.

### Step 5 — Main-agent loop (`orchestrator.py::run_main_agent`)

```
Turn 1:
  system prompt  ← MainAgentPromptBoxedAnswer.generate_system_prompt(tools)
  user message   ← task description + file attachment (if any)
  LLM response   ← "I will read the file using the read_file tool."
                   + tool_call: {server="tool-reading", tool="read_file", args={path=...}}

Turn 2:
  tool result    ← contents of the XLSX file (returned by tool-reading MCP server)
  LLM response   ← "\boxed{Congo Democratic Republic}"
                   stop_reason = "end_turn"  → loop exits
```

The loop runs until:
- The LLM produces a response with no tool calls (`stop_reason = "end_turn"`), **or**
- `max_turns` is reached (default: `-1` = unlimited).

### Step 6 — Answer extraction

`extract_gaia_final_answer()` (in `src/utils/summary_utils.py`) extracts the content inside `\boxed{...}` from the final response.

### Step 7 — Logging and cleanup

`TaskTracer.save()` writes the full structured log (all messages, tool calls, timings, token counts) to the log file. LLM client connections are closed.

---

## 5. The Configuration System (YAML + .env)

MiroFlow uses [Hydra](https://hydra.cc/) for configuration. Everything is controlled through YAML files in the `config/` directory.

### 5.1 The .env file

Copy `.env.template` to `.env` and fill in your keys:

```bash
# Minimum required to run quickstart examples
OPENROUTER_API_KEY="sk-or-v1-..."

# Required for web-search tools
SERPER_API_KEY="..."
JINA_API_KEY="..."

# Required for code execution tool
E2B_API_KEY="..."

# Required if using Anthropic API directly
ANTHROPIC_API_KEY="..."

# Required if using OpenAI API directly
OPENAI_API_KEY="..."

# Optional: override data directory
DATA_DIR="data"

# Optional: enable Chinese language prompts
CHINESE_CONTEXT="false"
```

### 5.2 Agent YAML anatomy

Every agent config file follows this structure:

```yaml
defaults:
  - benchmark: example_dataset    # merge config/benchmark/example_dataset.yaml
  - override hydra/job_logging: none
  - _self_                        # variables in this file win over defaults

main_agent:
  prompt_class: MainAgentPromptBoxedAnswer  # Python class in config/agent_prompts/

  llm:
    provider_class: "ClaudeOpenRouterClient"    # class in src/llm/providers/
    model_name: "anthropic/claude-3.7-sonnet"
    temperature: 0.3
    max_tokens: 32000
    openrouter_api_key: "${oc.env:OPENROUTER_API_KEY,???}"  # read from .env

  tool_config:           # which tools this agent has access to
    - tool-reading       # → loads config/tool/tool-reading.yaml

  max_turns: -1          # -1 = no limit
  max_tool_calls_per_turn: 10

  input_process:
    hint_generation: false       # use LLM to generate task hints before solving
  output_process:
    final_answer_extraction: false  # use LLM to extract final answer from response

  add_message_id: true           # add unique IDs to messages (avoids cache collisions)
  keep_tool_result: -1           # how many past tool results to keep in context (-1 = all)
  chinese_context: "${oc.env:CHINESE_CONTEXT,false}"

sub_agents:
  agent-worker:                  # name of this sub-agent
    prompt_class: SubAgentWorkerPrompt
    llm:
      provider_class: "ClaudeOpenRouterClient"
      model_name: "anthropic/claude-3.7-sonnet"
    tool_config:
      - tool-searching
      - tool-reading
      - tool-code
    max_turns: -1

# null means no sub-agents (single-agent mode):
# sub_agents: null

output_dir: logs/               # where to save logs
data_dir: "${oc.env:DATA_DIR,data}"
```

### 5.3 Tool config files

Each tool is a one-to-one mapping between a short name and an MCP server process:

```yaml
# config/tool/tool-searching.yaml
name: "tool-searching"
tool_command: "python"
args:
  - "-m"
  - "src.tool.mcp_servers.searching_mcp_server"
env:
  SERPER_API_KEY: "${oc.env:SERPER_API_KEY}"
  JINA_API_KEY: "${oc.env:JINA_API_KEY}"
```

When `tool_config: [tool-searching]` is set in an agent, MiroFlow launches `python -m src.tool.mcp_servers.searching_mcp_server` as a child process and communicates with it over stdio using the MCP protocol.

### 5.4 Benchmark config files

```yaml
# config/benchmark/gaia-validation.yaml (example)
name: "gaia-validation"
data:
  data_dir: "${data_dir}/gaia/validation"
execution:
  max_tasks: null       # null = run all
  max_concurrent: 3     # parallel tasks
  pass_at_k: 1          # attempts per task
```

---

## 6. LLM Clients — How MiroFlow Talks to AI Models

All LLM clients live in `src/llm/providers/` and inherit from `LLMProviderClientBase` (`src/llm/provider_client_base.py`).

### 6.1 How the factory works

`src/llm/client.py` exports a `LLMClient()` *factory function* (not a class). It reads `provider_class` from the config and dynamically imports the matching class:

```python
# config says: provider_class: "ClaudeOpenRouterClient"
providers_module = importlib.import_module("src.llm.providers")
ProviderClass = getattr(providers_module, "ClaudeOpenRouterClient")
return ProviderClass(task_id=task_id, task_log=task_log, cfg=config)
```

### 6.2 Available providers

| Config value | File | API used | Key(s) needed |
|---|---|---|---|
| `ClaudeOpenRouterClient` | `claude_openrouter_client.py` | OpenRouter (OpenAI-compatible) | `OPENROUTER_API_KEY` |
| `ClaudeAnthropicClient` | `claude_anthropic_client.py` | Anthropic official SDK | `ANTHROPIC_API_KEY` |
| `GPTOpenAIClient` | `gpt_openai_client.py` | OpenAI | `OPENAI_API_KEY` |
| `GPT5OpenAIClient` | `gpt5_openai_client.py` | OpenAI | `OPENAI_API_KEY` |
| `DeepSeekOpenRouterClient` | `deepseek_openrouter_client.py` | OpenRouter | `OPENROUTER_API_KEY` |
| `MiroThinkerSGLangClient` | `mirothinker_sglang_client.py` | Local SGLang server | none (self-hosted) |

### 6.3 What every client must implement

```python
class LLMProviderClientBase(ABC):
    async def _create_message(self, system_prompt, messages, tools_definitions, ...):
        """Send a single chat completion request. Return raw API response."""

    def process_llm_response(self, response, message_history, agent_type):
        """Extract text + tool calls from raw response, append to history."""
        # Returns (assistant_text, should_break)

    def extract_tool_calls_info(self, response, assistant_response_text):
        """Return structured tool-call info (or None)."""

    def close(self):
        """Clean up any HTTP connections."""
```

### 6.4 How tool calls flow through the LLM client

1. `Orchestrator` calls `llm_client.create_message(system_prompt, message_history, tool_definitions)`.
2. The client sends the request to the external API with the tool schemas serialised as JSON.
3. The API response may contain text *and/or* a `tool_calls` array.
4. `process_llm_response()` appends an `assistant` message (with the tool calls) to `message_history`.
5. Control returns to `Orchestrator`, which executes each requested tool call and appends a `tool` result message.
6. The loop calls `create_message()` again with the updated history.

---

## 7. Tools — What the Agent Can Do

### 7.1 How MCP works (plain English)

MCP (Model Context Protocol) is a standard protocol for "tool servers". Each tool in MiroFlow is a tiny Python server that:

1. Exposes a list of callable functions (its *tools*)
2. Waits on stdin for JSON-encoded call requests
3. Writes JSON-encoded results to stdout

`ToolManager` launches these servers as child processes and talks to them over stdio using the `mcp` Python library.

### 7.2 Tool catalogue

| Tool name in YAML | MCP server file | What it does | API key(s) needed |
|---|---|---|---|
| `tool-reading` | `reading_mcp_server.py` | Reads PDF, XLSX, DOCX, CSV, images, and web pages | `JINA_API_KEY` (optional, for web) |
| `tool-searching` | `searching_mcp_server.py` | Google search via Serper + web scraping via Jina | `SERPER_API_KEY`, `JINA_API_KEY` |
| `tool-searching-serper` | `miroapi_serper_mcp_server.py` | Alternative search endpoint (MiroAPI or Serper) | `SERPER_API_KEY` |
| `tool-code` | `python_server.py` | Execute Python in an E2B cloud sandbox | `E2B_API_KEY` |
| `tool-reasoning` | `reasoning_mcp_server.py` | Ask a powerful LLM to reason about a problem | `OPENROUTER_API_KEY` |
| `tool-reasoning-os` | `reasoning_mcp_server_os.py` | Same but using a local open-source model | SGLang server |
| `tool-image-video` | `vision_mcp_server.py` | Describe images/video frames using a vision LLM | `OPENROUTER_API_KEY` |
| `tool-image-video-os` | `vision_mcp_server_os.py` | Same but using a local model | SGLang server |
| `tool-audio` | `audio_mcp_server.py` | Transcribe audio files | `OPENROUTER_API_KEY` |
| `tool-audio-os` | `audio_mcp_server_os.py` | Same but using a local Whisper model | local Whisper |
| `tool-browsing` | *(playwright)* | Full browser automation via Playwright | none (headless browser) |
| `tool-markitdown` | *(markitdown)* | Convert various file formats to Markdown | none |

### 7.3 Searching tool — functions exposed to the LLM

| Function name | What it does |
|---|---|
| `google_search(query, num_results)` | Returns Google search result snippets |
| `scrape(url)` | Fetches and converts a web page to Markdown |
| `wikipedia_search(query)` | Searches Wikipedia |

### 7.4 Reading tool — functions exposed to the LLM

| Function name | What it does |
|---|---|
| `read_file(file_path)` | Reads a local file (PDF, XLSX, DOCX, CSV, TXT, …) |
| `scrape(url)` | Fetches a URL and returns Markdown |

---

## 8. External APIs You Need

| Service | Purpose | Free tier? | Sign-up URL |
|---|---|---|---|
| **OpenRouter** | Access Claude, GPT, DeepSeek, and 100+ models through a single endpoint | Yes (limited) | https://openrouter.ai/ |
| **Anthropic** | Direct access to Claude (alternative to OpenRouter) | No | https://console.anthropic.com/ |
| **OpenAI** | Direct access to GPT-4o, GPT-5 | Pay-per-use | https://platform.openai.com/ |
| **Serper** | Google Search API (for the web search tool) | 2,500 free queries/month | https://serper.dev/ |
| **Jina AI** | Web scraping/reader API (for fetching web content) | Free tier available | https://jina.ai/reader/ |
| **E2B** | Cloud Python sandboxes (for the code execution tool) | Free tier available | https://e2b.dev/ |

!!! tip "Minimum to get started"
    You only need an `OPENROUTER_API_KEY` to run the quickstart document-reading example. Add `SERPER_API_KEY` + `JINA_API_KEY` to unlock web search.

---

## 9. Command Reference

All commands are run via `uv run main.py <command> [options]`.

### `trace` — run a single task

```bash
uv run main.py trace \
  --config_file_name=<yaml_name>   \  # e.g. agent_quickstart_reading
  --task="<task description>"      \  # what you want the agent to do
  --task_file_name="<path>"        \  # optional: attach a file
  --task_id="my_task_001"             # optional: custom ID for the log file
```

### `common-benchmark` — run a full benchmark

```bash
uv run main.py common-benchmark \
  --config_file_name=agent_gaia-validation \
  output_dir="logs/gaia/$(date +%Y%m%d_%H%M)"
```

### `eval-answer` — score answers already stored in log files

```bash
uv run main.py eval-answer --log_dir="logs/gaia/20250101_1200"
```

### `avg-score` — compute average score across a run

```bash
uv run main.py avg-score --log_dir="logs/gaia/20250101_1200"
```

### `score-from-log` — alternative scoring helper

```bash
uv run main.py score-from-log --log_dir="logs/gaia/20250101_1200"
```

### `prepare-benchmark` — download and pre-process datasets

```bash
uv run main.py prepare-benchmark --benchmark=gaia
```

### `print-config` — debug your YAML config

```bash
uv run main.py print-config --config_file_name=agent_quickstart_reading
```

---

## 10. Running Your First Task (Hands-on)

### Prerequisites

```bash
# Python 3.12+
python --version

# Install uv (package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone the repo
git clone https://github.com/MiroMindAI/MiroFlow && cd MiroFlow

# Install all dependencies
uv sync
```

### Set up your API key

```bash
cp .env.template .env
# Open .env in your editor and set:
# OPENROUTER_API_KEY="sk-or-v1-..."
```

### Run Example 1: Analyse a spreadsheet

```bash
uv run main.py trace \
  --config_file_name=agent_quickstart_reading \
  --task="What is the first country listed in the XLSX file whose name starts with Co?" \
  --task_file_name="data/FSI-2023-DOWNLOAD.xlsx"
```

**Expected output:** `\boxed{Congo Democratic Republic}`

### Run Example 2: Search the web

```bash
# Also add SERPER_API_KEY and JINA_API_KEY to .env first
uv run main.py trace \
  --config_file_name=agent_quickstart_search \
  --task="What is the current NASDAQ index price and what factors affect it today?"
```

### Run Example 3: Single agent with web search (no sub-agent)

```bash
uv run main.py trace \
  --config_file_name=agent_quickstart_single_agent \
  --task="Find the population of Tokyo as of the latest available data."
```

---

## 11. Understanding the Logs

After a run, MiroFlow writes a structured `.log` file to the `output_dir` you specified (default: `logs/`).

```json
{
  "task_id": "task_1",
  "task_name": "task_1",
  "status": "completed",
  "start_time": "2025-01-01T12:00:00",
  "end_time": "2025-01-01T12:01:23",
  "input": {
    "task_description": "What is the first country …",
    "task_file_name": "data/FSI-2023-DOWNLOAD.xlsx"
  },
  "final_boxed_answer": "Congo Democratic Republic",
  "steps": [
    {"step_id": "task_execution_started", "message": "…"},
    {"step_id": "main_agent_turn_1_llm_call_success", "message": "…"},
    {"step_id": "tool_call_read_file", "message": "…"},
    {"step_id": "task_execution_finished", "message": "…"}
  ],
  "main_agent_message_history": {
    "system_prompt": "…",
    "message_history": [ … ]
  }
}
```

**Key fields:**

| Field | Meaning |
|---|---|
| `status` | `running` → `completed` or `interrupted` |
| `final_boxed_answer` | The extracted answer (if `\boxed{}` was used) |
| `steps` | Chronological list of all events during the run |
| `main_agent_message_history` | Full conversation (system prompt + all turns) |
| `sub_agent_message_history_sessions` | Same, but for each sub-agent session |

---

## 12. Running Benchmarks

Benchmarks are multi-task runs where the agent solves a standardised dataset and results are scored automatically.

### Quick benchmark run (GAIA validation)

```bash
# 1. Download the dataset first
uv run main.py prepare-benchmark --benchmark=gaia

# 2. Run the benchmark
uv run main.py common-benchmark \
  --config_file_name=agent_gaia-validation_claude37sonnet \
  output_dir="logs/gaia/$(date +%Y%m%d_%H%M)"

# 3. Compute the score
uv run main.py avg-score --log_dir="logs/gaia/<timestamp>"
```

### Concurrency

Set `execution.max_concurrent` in the benchmark YAML (e.g. `config/benchmark/gaia-validation.yaml`) to run multiple tasks in parallel. Default is `3`.

### Supported benchmarks

| Benchmark | Config file prefix | What it tests |
|---|---|---|
| GAIA Validation | `agent_gaia-validation` | General AI assistant tasks |
| GAIA Test | `agent_gaia-test` | Hidden test set |
| HLE | `agent_hle` | Humanity's Last Exam (multimodal) |
| HLE Text-Only | `agent_hle-text-only` | HLE without images |
| BrowseComp-EN | `agent_browsecomp-en` | English web browsing QA |
| BrowseComp-ZH | `agent_browsecomp-zh` | Chinese web browsing QA |
| WebWalkerQA | `agent_webwalkerqa` | Web traversal QA |
| xBench-DS | `agent_xbench-ds` | Deep search |
| FinSearchComp | `agent_finsearchcomp` | Financial search |
| FutureX | (see `futurex.md`) | Future event prediction |

---

## 13. How to Extend MiroFlow

### Add a new LLM provider

1. Create `src/llm/providers/my_new_client.py` that inherits `LLMProviderClientBase`.
2. Implement `_create_client()`, `_create_message()`, `process_llm_response()`, `extract_tool_calls_info()`.
3. Export it in `src/llm/providers/__init__.py`.
4. Use `provider_class: "MyNewClient"` in your YAML config.

See [Contributing LLM Clients](contribute_llm_clients.md) for details.

### Add a new tool

1. Create `src/tool/mcp_servers/my_tool_server.py` using the `fastmcp` library.
2. Create `config/tool/my-tool.yaml` pointing to your new server.
3. Reference `my-tool` in any agent's `tool_config:` list.

See [Contributing Tools](contribute_tools.md) for details.

### Add a new benchmark

1. Create `config/benchmark/my-benchmark.yaml` with dataset path and execution settings.
2. Create the agent config referencing your benchmark and desired tools.
3. Ensure your dataset is in the expected format (see `utils/prepare_benchmark/`).

See [Contributing Benchmarks](contribute_benchmarks.md) for details.

---

## 14. Glossary

| Term | Definition |
|---|---|
| **Agent** | An AI model that can take multi-step actions using tools to complete a task |
| **Main Agent** | The top-level agent that plans and delegates work |
| **Sub-Agent** | A specialised agent called by the main agent to execute a subtask |
| **MCP** | Model Context Protocol — a standard for connecting AI models to external tools |
| **MCP Server** | A small process that exposes a set of callable functions (tools) over MCP |
| **Tool** | A callable function an agent can invoke (e.g. `google_search`, `read_file`) |
| **ToolManager** | MiroFlow component that manages connections to MCP servers |
| **LLMClient** | MiroFlow abstraction over a specific LLM provider API |
| **Orchestrator** | The core loop that interleaves LLM calls and tool executions |
| **Pipeline** | Wires together LLMClient, ToolManager, Orchestrator for one task |
| **TaskTracer** | Writes a structured log of every step in a task run |
| **Hydra** | Python configuration library used to load and merge YAML files |
| **OpenRouter** | A unified API that routes requests to many different LLM providers |
| **SGLang** | A serving framework for running open-source LLMs locally |
| **E2B** | Cloud sandbox service used to safely execute Python code |
| **Serper** | Google Search API service |
| **Jina** | Web reader/scraping API service |
| **Boxed answer** | The final answer wrapped in `\boxed{…}`, a convention borrowed from math olympiads |
| **GAIA** | *General AI Assistants* benchmark — a suite of tasks requiring tool use |
| **HLE** | *Humanity's Last Exam* — an extremely hard multimodal benchmark |
| **BrowseComp** | A web-browsing QA benchmark from OpenAI |
| `uv` | A fast Python package manager used in this project |

---

!!! info "Documentation Info"
    **Last Updated:** 2026 · **Doc Contributor:** MiroFlow Community · **Version:** v0.3
