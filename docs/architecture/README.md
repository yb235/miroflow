# MiroFlow Architecture Guide

> **Who is this for?** Anyone — from first-time contributors to experienced engineers — who wants to understand how MiroFlow works under the hood. Each page is self-contained, so you can read them in any order.

---

## What is MiroFlow in one sentence?

MiroFlow is a **modular research-agent framework** that wires together large language models (LLMs), a set of tools (search, file reading, code execution, etc.), and a control loop (the "orchestrator") to solve complex, multi-step tasks — such as predicting future events or answering tough benchmark questions.

---

## How to read these docs

| If you want to… | Start here |
|-----------------|------------|
| Get the big picture | [01 — High-Level Overview](01-overview.md) |
| Know where files live | [02 — Directory Structure](02-directory-structure.md) |
| Run the CLI commands | [03 — Entry Points & CLI](03-entry-points-and-cli.md) |
| Understand configuration | [04 — Configuration System](04-configuration-system.md) |
| Understand how agents work | [05 — Agent Architecture](05-agent-architecture.md) |
| Dive into the core loop | [06 — Orchestrator](06-orchestrator.md) |
| See how components are assembled | [07 — Pipeline Layer](07-pipeline.md) |
| Learn about LLM provider abstraction | [08 — LLM Client](08-llm-client.md) |
| Explore the tool/MCP system | [09 — Tool System](09-tool-system.md) |
| Trace a request end-to-end | [10 — Data Flow](10-data-flow.md) |
| Look up data models | [11 — Data Schemas](11-data-schemas.md) |
| Understand logging | [12 — Logging & Observability](12-logging.md) |
| Run benchmarks | [13 — Benchmark Evaluation](13-benchmarks.md) |
| Learn about input/output handling | [14 — Input/Output Processing](14-input-output.md) |
| Understand *why* things are designed this way | [15 — Design Decisions & Rationale](15-design-decisions.md) |
| Find a specific class or function | [16 — API Reference](16-api-reference.md) |
| See what libraries are used | [17 — Dependencies](17-dependencies.md) |

---

## Quick Architecture Diagram

```
User (CLI)
   │
   ▼
Configuration (Hydra + YAML)
   │
   ▼
Pipeline (assembles components)
   │
   ├──▶ LLM Client (talks to OpenAI / Anthropic / etc.)
   ├──▶ Tool Manager (connects to MCP tool servers)
   └──▶ Output Formatter
         │
         ▼
   Orchestrator (the brain — runs the agent loop)
      │
      ├──▶ Main Agent  ──calls──▶ tools / sub-agents
      └──▶ Sub-Agent   ──calls──▶ search, read, code, vision…
         │
         ▼
   Final Answer (\boxed{…})
```

> **Design rationale:** Splitting the system into these layers (CLI → Config → Pipeline → Orchestrator → LLM/Tools) makes each part independently testable and replaceable. You can swap the LLM provider without touching the orchestrator, or add a new tool without changing the pipeline logic.

---

*For the original monolithic document, see [ARCHITECTURE.md](../ARCHITECTURE.md). These individual pages break down the same content into smaller, easier-to-digest pieces with added design rationale.*
