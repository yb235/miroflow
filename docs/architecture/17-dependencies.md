# 17 — Dependencies

> **In plain English:** This page lists all the major libraries MiroFlow depends on and explains what each one is used for. If you see an unfamiliar `import` in the code, check here.

---

## Core Framework

| Package | What it does | Why MiroFlow uses it |
|---------|-------------|---------------------|
| `fire` | Turns Python functions into CLI commands | Powers the `main.py` command-line interface with near-zero boilerplate |
| `hydra-core` / `omegaconf` | Configuration management with YAML | Enables reproducible, composable experiment configs |
| `pydantic` | Data validation with type hints | Ensures structured logs (TaskTracer) are always valid |
| `rich` | Beautiful terminal output | Color-coded logs, progress bars, formatted tracebacks |

## LLM Providers

| Package | What it does | Why MiroFlow uses it |
|---------|-------------|---------------------|
| `openai` | OpenAI API client | Talks to GPT models (also used for OpenRouter) |
| `anthropic` | Anthropic API client | Talks to Claude models directly |
| `google-genai` | Google Gemini API client | Vision tool uses Gemini as a fallback provider |
| `tiktoken` | Token counting | Estimates context length to manage the context window |
| `tenacity` | Retry logic with exponential backoff | Handles transient API failures (rate limits, timeouts) gracefully |

## Tool System

| Package | What it does | Why MiroFlow uses it |
|---------|-------------|---------------------|
| `mcp` / `fastmcp` | Model Context Protocol | Standard protocol for connecting agents to tool servers |
| `e2b-code-interpreter` | Sandboxed Python execution | Runs LLM-generated code safely in an isolated cloud environment |
| `aiohttp` | Async HTTP client | Tool servers need fast, non-blocking HTTP for web search/scraping |
| `wikipedia` | Wikipedia API client | The search tool can fetch Wikipedia articles |
| `json5` | Lenient JSON parsing | LLMs sometimes produce invalid JSON; json5 is more forgiving |
| `markitdown-mcp` | Document-to-markdown conversion | Alternative document reading tool |

## Logging

| Package | What it does | Why MiroFlow uses it |
|---------|-------------|---------------------|
| `pyzmq` | ZeroMQ messaging | Cross-process log aggregation from MCP server processes |

## Data & File Processing

| Package | What it does | Why MiroFlow uses it |
|---------|-------------|---------------------|
| `datasets` | HuggingFace datasets loader | Loads benchmark datasets for evaluation |
| `pandas` / `openpyxl` | Data manipulation & Excel support | The reading tool processes spreadsheets and tabular data |
| `Pillow` | Image processing | Vision tools process and resize images |
| `mutagen` | Audio file metadata | Audio tool reads file metadata to handle different formats |

## Development & Environment

| Package | What it does | Why MiroFlow uses it |
|---------|-------------|---------------------|
| `dotenv` | Load `.env` files | Reads API keys from `.env` into environment variables |
| `uv` | Fast Python package manager | Used instead of pip for faster dependency resolution |

---

> **Design rationale — Dependency choices:**
> - **Established libraries** (openai, anthropic, pydantic) are preferred over niche alternatives for better documentation, community support, and long-term maintenance.
> - **MCP** was chosen as the tool protocol because it's an open standard backed by Anthropic, ensuring broad compatibility.
> - **E2B** was chosen for code execution because it provides secure sandboxing without requiring complex local Docker setups.
> - **ZMQ** was chosen for logging because it's the lightest-weight option for cross-process messaging with real-time delivery.

---

**Previous:** [16 — API Reference](16-api-reference.md) · **Back to index:** [README](README.md)
