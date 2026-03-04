# 02 — Directory Structure

> **In plain English:** This page shows you where everything lives in the repository. If you're looking for a specific file, start here.

---

## Top-Level Layout

```
miroflow/
├── main.py                     ← The front door — you run this to use MiroFlow
├── common_benchmark.py         ← Runs batch benchmark evaluations
├── pyproject.toml              ← Project metadata & Python dependencies
├── .env.template               ← Template for your API keys
│
├── config/                     ← All configuration lives here (YAML files)
├── src/                        ← All source code lives here
├── utils/                      ← Evaluation & utility scripts
├── scripts/                    ← Shell scripts for batch runs
├── data/                       ← Sample data files
└── docs/                       ← Documentation (you are here!)
```

---

## `config/` — Configuration

```
config/
├── __init__.py                  ← Helpers: config_path(), config_name(), debug_config()
├── agent_*.yaml                 ← One file per agent setup (defines LLM + tools + prompts)
├── agent_prompts/               ← Python prompt classes (generate system prompts)
│   ├── base_agent_prompt.py     ← Abstract base class — all prompts inherit from this
│   ├── main_agent_prompt_gaia.py
│   ├── main_agent_prompt_deepseek.py
│   ├── main_boxed_answer.py
│   └── sub_worker.py
├── benchmark/                   ← Benchmark dataset configs (GAIA, HLE, etc.)
└── tool/                        ← Tool server configs (one YAML per tool)
```

> **Design rationale — Config as YAML:** Human-readable YAML files make it easy to create new agent configurations without writing any code. Want to try a different LLM model? Just duplicate a YAML file and change the `model_name` field. This "configuration-driven" approach means experimentation is fast and doesn't risk breaking source code.

---

## `src/` — Source Code

```
src/
├── core/                        ← The brain of the system
│   ├── orchestrator.py          ← Main + sub-agent loops, tool dispatch
│   └── pipeline.py              ← Pipeline assembly, task execution entry point
│
├── llm/                         ← Language model integration
│   ├── client.py                ← Factory function — picks the right LLM provider
│   ├── provider_client_base.py  ← Abstract base class for all providers
│   ├── util.py                  ← Timeout decorator
│   └── providers/               ← One file per LLM provider
│       ├── claude_openrouter_client.py
│       ├── claude_anthropic_client.py
│       ├── gpt_openai_client.py
│       ├── gpt5_openai_client.py
│       ├── deepseek_openrouter_client.py
│       └── mirothinker_sglang_client.py
│
├── tool/                        ← Tool system (MCP servers)
│   ├── manager.py               ← ToolManager: connects to and runs tools
│   └── mcp_servers/             ← Individual tool implementations
│       ├── searching_mcp_server.py   ← Google/Serper search + Jina scraping
│       ├── reading_mcp_server.py     ← File reading (PDF, XLSX, DOCX, etc.)
│       ├── reasoning_mcp_server.py   ← Pure text reasoning via LLM
│       ├── vision_mcp_server.py      ← Image/video analysis
│       ├── audio_mcp_server.py       ← Audio transcription
│       ├── python_server.py          ← E2B sandboxed Python execution
│       ├── browser_session.py        ← Persistent Playwright browser session
│       └── utils/
│           ├── smart_request.py      ← Smart HTTP with fallbacks
│           └── url_unquote.py
│
├── utils/                       ← Shared utility functions
│   ├── io_utils.py              ← Input processing, output formatting, \boxed{} extraction
│   ├── parsing_utils.py         ← JSON repair, XML tool-call parsing
│   ├── summary_utils.py         ← Hint generation, final answer extraction
│   └── tool_utils.py            ← MCP server param creation, sub-agent exposure
│
└── logging/                     ← Logging infrastructure
    ├── logger.py                ← Logger setup, ZMQ handler/listener
    └── task_tracer.py           ← Pydantic TaskTracer model (structured logs)
```

> **Design rationale — Flat `src/` with clear folders:** Each subfolder (`core/`, `llm/`, `tool/`, `utils/`, `logging/`) represents a distinct concern. This means a developer working on tool integration never needs to touch the LLM code, and vice versa. The flat structure (no deeply nested folders) keeps navigation simple.

---

## `utils/` — Evaluation & Utility Scripts

```
utils/
├── eval_utils.py                ← Shared evaluation helpers
├── eval_answer_from_log.py      ← Evaluate answers from log files
├── calculate_average_score.py   ← Calculate average benchmark scores
├── calculate_score_from_log.py  ← Score from existing log files
├── trace_single_task.py         ← Run a single task (used by CLI)
└── prepare_benchmark/           ← Download and prepare benchmark datasets
```

> **Design rationale — Separate `utils/` from `src/`:** The top-level `utils/` contains scripts that are run directly (via CLI) for evaluation and benchmarking. They're separate from `src/` because they're "consumers" of the framework, not part of the core library. This distinction helps contributors understand what's library code vs. what's tooling.

---

**Next:** [03 — Entry Points & CLI](03-entry-points-and-cli.md) · **Previous:** [01 — Overview](01-overview.md)
