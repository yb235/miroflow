# 12 — Logging & Observability

> **In plain English:** When MiroFlow runs, multiple processes are working at the same time (the main agent, tool servers, etc.). This page explains how all their logs are collected into one place, and how task execution is tracked for debugging.

---

## The Problem: Multi-Process Logging

MiroFlow's tool servers run as **separate processes** (each MCP server is its own Python process). Standard Python logging only works within a single process. So how do we see logs from all the tool servers?

**Answer: ZeroMQ (ZMQ)** — a lightweight messaging library.

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

**How it works:**
1. Each MCP server process has a `ZMQHandler` that *pushes* log messages to a ZMQ socket.
2. The main MiroFlow process runs a `ZMQListener` that *pulls* messages from all servers.
3. The listener forwards those messages to the standard Python logger.
4. Result: all logs appear in one unified stream.

**Message format:** `{task_id}||{tool_name}||{message}`

**Port management:** Automatically finds an available port starting from 6000, with fallback to random port binding.

**File:** `src/logging/logger.py`

> **Design rationale — ZMQ for cross-process logging:**
> 1. **Lightweight:** ZMQ is fast and has minimal overhead — important when you're running many tool servers concurrently.
> 2. **No shared filesystem needed:** Unlike file-based logging, ZMQ works across processes without file locking issues.
> 3. **Real-time:** Logs appear immediately (no waiting for file flushes).
> 4. **Alternatives considered:** Python's `logging.handlers.SocketHandler` works but requires more setup. ZMQ's PUSH/PULL pattern is simpler and more robust for this use case.

---

## Task Context Variables

`TASK_CONTEXT_VAR` is a Python `ContextVar` that stores the current task ID. This enables three features:

1. **Per-task log files:** A `TaskFilter` routes logs to the correct task-specific log file.
2. **Automatic task ID in logs:** A custom `LogRecordFactory` injects the task ID into every log record.
3. **Task ID propagation to tool servers:** The task ID is passed to MCP servers via environment variables.

> **Design rationale — ContextVar for task tracking:** When running multiple tasks concurrently (as in benchmark evaluation), `ContextVar` ensures each async task sees its own task ID — even though they share the same Python process. This is asyncio-native and doesn't require threading locks.

---

## Console Output

MiroFlow uses [Rich](https://rich.readthedocs.io/) for beautiful console output with:
- Color-coded log levels.
- Full tracebacks with local variable display.
- Progress indicators.

The logger is bootstrapped with `bootstrap_logger(level=LOGGER_LEVEL)`.

---

## Summary

| Component | Purpose | File |
|-----------|---------|------|
| `bootstrap_logger()` | Sets up Rich console + file logging | `src/logging/logger.py` |
| `ZMQHandler` | Pushes logs from MCP servers | `src/logging/logger.py` |
| `ZMQListener` | Pulls logs in the main process | `src/logging/logger.py` |
| `setup_mcp_logging()` | Configures ZMQ logging for MCP processes | `src/logging/logger.py` |
| `TASK_CONTEXT_VAR` | Per-task context tracking | `src/logging/logger.py` |
| `TaskTracer` | Structured Pydantic task log | `src/logging/task_tracer.py` |

---

**Next:** [13 — Benchmark Evaluation](13-benchmarks.md) · **Previous:** [11 — Data Schemas](11-data-schemas.md)
