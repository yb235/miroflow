# 01 — High-Level Overview

> **In plain English:** MiroFlow takes a question from you, hands it to an AI "agent," and that agent uses a mix of thinking (LLM calls) and doing (tool calls — like searching the web or reading a file) to produce an answer. Think of it like a very capable research assistant that can Google things, read PDFs, run code, and analyze images — all on its own.

---

## The Seven Layers

MiroFlow is organized into seven layers, stacked on top of each other like a cake. Each layer has one job:

| # | Layer | What it does | Key files |
|---|-------|-------------|-----------|
| 1 | **CLI** | Accepts your commands from the terminal | `main.py` |
| 2 | **Configuration** | Loads settings from YAML files | `config/` directory |
| 3 | **Pipeline** | Assembles all the pieces together | `src/core/pipeline.py` |
| 4 | **Orchestrator** | Runs the agent loop: think → act → observe → repeat | `src/core/orchestrator.py` |
| 5 | **LLM** | Talks to language model APIs (OpenAI, Anthropic, etc.) | `src/llm/` |
| 6 | **Tools** | Connects to MCP tool servers (search, file reading, etc.) | `src/tool/` |
| 7 | **Logging** | Records everything that happens for debugging | `src/logging/` |

### Why seven layers?

> **Design rationale:** Each layer is responsible for exactly one concern. This "separation of concerns" means you can:
> - Change how you talk to OpenAI *without* touching the orchestrator.
> - Add a brand-new tool *without* changing any LLM code.
> - Switch from one config format to another *without* rewriting the agent logic.
>
> This makes the codebase easier to maintain, test, and extend.

---

## Architecture Diagram

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

---

## The Core Idea: Think → Act → Observe → Repeat

The heart of MiroFlow is a simple loop:

1. **Think** — The LLM reads the conversation so far and decides what to do next.
2. **Act** — If the LLM wants to use a tool (like searching the web), MiroFlow executes that tool.
3. **Observe** — The tool's result is added to the conversation.
4. **Repeat** — Go back to step 1, until the LLM produces a final answer.

> **Design rationale:** This "ReAct" loop (Reasoning + Acting) is the standard pattern in modern agent systems. It allows the agent to gather information iteratively, just like a human researcher would. The loop continues until the agent has enough information to answer confidently, or until a safety limit (`max_turns`) is reached.

---

**Next:** [02 — Directory Structure](02-directory-structure.md)
