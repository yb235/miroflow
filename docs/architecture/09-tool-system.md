# 09 — Tool System (MCP Servers)

> **In plain English:** Tools are what give the agent its "superpowers" — the ability to search the web, read files, run code, and more. Each tool runs as a separate little server (an MCP server), and the `ToolManager` connects to them. This page explains the whole tool system.

---

## What is MCP?

**MCP** stands for **Model Context Protocol**. It's a standard way for AI agents to talk to tool servers. Think of it like USB for AI tools — any tool that speaks MCP can plug into any agent that supports MCP.

Each MCP server is a **separate process** that exposes one or more tools. Communication happens over **stdio** (standard input/output) or **SSE** (Server-Sent Events).

> **Design rationale — Why MCP?**
> 1. **Isolation:** Each tool runs in its own process. A crash in the search tool doesn't bring down the entire agent.
> 2. **Language-agnostic:** MCP servers can be written in any language (MiroFlow's are in Python, but they don't have to be).
> 3. **Standardization:** MCP is an open standard. You can use third-party MCP servers (like `markitdown-mcp`) alongside custom ones.
> 4. **Security:** Process isolation means a tool can't access the agent's memory or other tools' data.

---

## ToolManager

**File:** `src/tool/manager.py`

The `ToolManager` is the bridge between the agent and the tool servers:

```python
class ToolManager:
    async def get_all_tool_definitions() -> list[dict]    # What tools are available?
    async def execute_tool_call(server, tool, args) -> dict  # Run a specific tool
```

**Key features:**
- **900-second timeout** on tool execution (prevents hanging).
- **Tool blacklisting** — certain (server, tool) pairs can be excluded.
- **Persistent browser session** — the Playwright browser stays alive across tool calls (avoids slow reconnection).

> **Design rationale — Central ToolManager:** Having one manager for all tools gives the system a single place to handle timeouts, error recovery, and tool discovery. The agent never talks to tool servers directly — it always goes through the ToolManager.

---

## Available Tool Servers

| Tool | What it does | Key tools exposed |
|------|-------------|-------------------|
| **Searching** | Web search and scraping | `google_search`, `google_news_search`, `scrape_website`, `get_wikipedia_article`, `get_calendar` |
| **Reading** | Read files (PDF, XLSX, DOCX, etc.) | `read_file` |
| **Reasoning** | Deep thinking via a separate LLM | `reasoning` |
| **Vision** | Analyze images and videos | `describe_image`, `visual_question_answering` |
| **Audio** | Transcribe audio files | `transcribe_audio` |
| **Python/Code** | Run Python in a sandbox | `execute_python`, `upload_file_to_sandbox`, `download_file_from_sandbox` |
| **Browser** | Web browsing with Playwright | (various Playwright tools) |

### Searching Server
- **Backends:** Serper API (Google search) + Jina AI (web scraping) + Wikipedia API.
- Configurable result filtering (you can remove snippets, knowledge graphs, answer boxes).

### Reading Server
- Handles: PDF, DOCX, XLSX, CSV, PPT, HTML, ZIP, TXT, JSON.
- Supports URI-based file access (`file:`, `data:`, `http:` schemes).
- Falls back to web scraping if direct reading fails.

### Reasoning Server
- A "think harder" tool — delegates complex analysis to a separate LLM (like OpenAI o3 or Claude).
- Use cases: complex math, puzzles, integrating information, planning.

### Vision Server
- Multi-provider: tries Claude → GPT-4o → Gemini based on available API keys.
- Supports both local files and URLs.

### Audio Server
- Uses OpenAI Whisper / GPT-4o audio.
- Handles multiple audio formats, with URL download retry.

### Python/Code Server
- Uses **E2B** (cloud sandbox) — not local execution.
- Pre-installed with dozens of data science packages.
- Includes file upload/download for working with data.

> **Design rationale — E2B sandboxed execution:** Running untrusted code locally would be a security risk. E2B provides an isolated cloud sandbox, so even if the LLM generates malicious code, it can't affect the host system.

### Browser Session
- Persistent Playwright MCP session — the browser stays open across tool calls.
- Supports both stdio and SSE connections.

> **Design rationale — Persistent browser session:** Opening a new browser for every tool call would be slow and would lose state (cookies, navigation history). A persistent session lets the agent browse naturally — click a link, read the page, click another link — just like a human would.

---

## Tool Execution Flow

When the LLM says "I want to use a tool," here's what happens:

```
LLM Response: "I'll search for this"
     │
     ▼
Parse tool calls (XML or OpenAI function calls)
Extract: server_name, tool_name, arguments
     │
     ▼
Is server_name "agent-*"? ──YES──▶ Spawn sub-agent loop (see 06-orchestrator.md)
     │ NO
     ▼
Is it "playwright"? ──YES──▶ Use persistent PlaywrightSession
     │ NO
     ▼
Is it a stdio server? ──YES──▶ Connect via stdio, call tool
     │ NO
     ▼
Is it SSE? ──YES──▶ Connect via SSE, call tool
```

---

## Auto-Correction & Error Handling

LLMs sometimes hallucinate tool server names (e.g., calling `"search"` instead of `"tool-searching"`). The ToolManager handles this:

1. Searches all servers for a tool matching the `tool_name`.
2. If found in **exactly one server** → auto-corrects and executes (adds a note to the result).
3. If found in **multiple servers** → returns an error listing the options.
4. If not found anywhere → returns a helpful error message.

Additional protections:
- **HuggingFace scraping protection** — blocks scraping of HuggingFace dataset pages to prevent the agent from "cheating" on benchmarks by directly reading ground truth data.
- **Tool result truncation** — results are truncated to 100,000 characters (~25k tokens) to prevent context overflow.

> **Design rationale — Auto-correction:** LLMs are imperfect and sometimes get tool names slightly wrong. Rather than failing hard, auto-correction gives the agent a second chance. This significantly reduces tool-call failures in practice and makes the agent more robust.

---

## Tool Configuration

Tools are configured in `config/tool/tool-*.yaml` and referenced by name in agent configs:

| Config File | Server Name | Python Module |
|-------------|-------------|---------------|
| `tool-searching.yaml` | `tool-searching` | `src.tool.mcp_servers.searching_mcp_server` |
| `tool-reading.yaml` | `tool-reading` | `src.tool.mcp_servers.reading_mcp_server` |
| `tool-reasoning.yaml` | `tool-reasoning` | `src.tool.mcp_servers.reasoning_mcp_server` |
| `tool-image-video.yaml` | `tool-image-video` | `src.tool.mcp_servers.vision_mcp_server` |
| `tool-audio.yaml` | `tool-audio` | `src.tool.mcp_servers.audio_mcp_server` |
| `tool-code.yaml` | `tool-code` | `src.tool.mcp_servers.python_server` |
| `tool-browsing.yaml` | `tool-browsing` | Playwright MCP |
| `tool-markitdown.yaml` | `tool-markitdown` | markitdown-mcp |

---

**Next:** [10 — Data Flow](10-data-flow.md) · **Previous:** [08 — LLM Client](08-llm-client.md)
