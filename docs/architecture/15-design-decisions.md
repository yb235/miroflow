# 15 — Design Decisions & Rationale

> **In plain English:** This page collects all the interesting architectural decisions in MiroFlow and explains *why* they were made. If you're wondering "why did they do it that way?", this is the page for you.

---

## 🧩 Sub-Agents as Tools

**What:** Sub-agents (like `agent-worker`) are exposed to the main agent as if they were regular tools. The main agent calls them with `<use_mcp_tool><server_name>agent-worker</server_name>...`.

**Why:**
- The main agent doesn't need special logic for delegation — it just "calls a tool."
- Adding new sub-agents is purely a config change.
- Sub-agents could theoretically delegate to *other* sub-agents, enabling recursive composition.
- The main agent focuses on *what* to do; the sub-agent focuses on *how* to do it.

**Where:** `tool_utils.py::expose_sub_agents_as_tools()`, `orchestrator.py`

---

## 📝 XML Tool Calling for Anthropic Models

**What:** Instead of using OpenAI's function-calling API, MiroFlow uses XML-formatted tool calls parsed with regex for Anthropic models.

**Why:**
- Gives MiroFlow full control over the tool-calling format.
- Works uniformly across providers without vendor lock-in.
- The XML format is explicit and easy to debug (you can read it in logs).
- For OpenAI models, native function calling is used because it's more reliable with those models.

**Where:** `config/agent_prompts/`, `src/utils/parsing_utils.py`

---

## 📨 Message ID Injection

**What:** Unique IDs like `[msg_a1b2c3d4]` are prepended to user messages.

**Why:**
- LLM providers cache responses based on conversation content.
- Without unique IDs, two different tasks with similar questions might get cached responses from each other.
- The unique ID ensures each conversation is treated as distinct by the provider's cache.

**Where:** Orchestrator, configured via `add_message_id: true`

---

## 🧠 Hint Generation as Pre-Processing

**What:** Before the main agent starts, a separate LLM call analyzes the question for subtle pitfalls.

**Why:**
- Benchmark questions (especially GAIA) are deliberately tricky — ambiguous wording, multiple valid interpretations.
- A cheap, focused "think about potential problems" step is much cheaper than having the agent fail and restart.
- The hints are added to the agent's context, making it more careful from the start.

**Where:** `src/utils/summary_utils.py::extract_hints()`

---

## 🔒 HuggingFace Scraping Protection

**What:** The system blocks attempts to scrape HuggingFace dataset pages.

**Why:**
- Benchmarks like GAIA store their ground truth on HuggingFace.
- Without this protection, the agent could "cheat" by directly reading the answers.
- This ensures benchmark results reflect genuine reasoning, not data leakage.

**Where:** `src/tool/manager.py`, `src/tool/mcp_servers/searching_mcp_server.py`

---

## 🌐 Cross-Process Logging with ZMQ

**What:** ZeroMQ (ZMQ) PUSH/PULL sockets aggregate logs from MCP server processes back to the main process.

**Why:**
- MCP tool servers are separate processes — regular Python logging doesn't work across process boundaries.
- ZMQ is lightweight, fast, and requires no shared filesystem.
- Logs appear in real-time (no waiting for file flushes).
- The PUSH/PULL pattern is simple and robust for this many-to-one use case.

**Where:** `src/logging/logger.py`

---

## 📊 Balanced-Brace `\boxed{}` Extraction

**What:** Instead of regex, a character-by-character brace-counting algorithm extracts `\boxed{}` content.

**Why:**
- Mathematical answers often have nested braces: `\boxed{f(x) = \frac{a}{b+c}}`.
- Regex would stop at the first closing brace, giving wrong results.
- Brace counting correctly matches the outermost braces regardless of nesting depth.

**Where:** `src/utils/io_utils.py::OutputFormatter._extract_boxed_content()`

---

## 🔄 Context Window Management

**What:** The `keep_tool_result` parameter and `_remove_tool_result_from_messages()` implement a sliding-window approach to manage context length.

**Why:**
- Long tasks accumulate many tool results, eventually exceeding the LLM's context window.
- Rather than failing, old tool results are replaced with `"Tool result is omitted to save tokens."`.
- The first and last N messages are always kept (the beginning has the task setup; the end has the latest results).

**Where:** `src/llm/provider_client_base.py`

---

## 🛡️ Auto-Correction of Tool Server Names

**What:** When the LLM calls a non-existent tool server, the `ToolManager` searches all servers for the tool and auto-corrects if there's an unambiguous match.

**Why:**
- LLMs sometimes hallucinate server names (e.g., `"search"` instead of `"tool-searching"`).
- Hard failure on every hallucinated name would waste entire turns of the agent loop.
- Auto-correction gives the agent a second chance, significantly reducing tool-call failures.

**Where:** `src/tool/manager.py`

---

## 🔄 Provider-Specific Role Handling

**What:** OpenAI's newer models (`o1`, `o3`, `o4`, `gpt-5`) use `developer` role instead of `system`, while older models and Anthropic use `system`.

**Why:**
- Each LLM provider has its own API conventions.
- By handling this in individual provider files, the orchestrator doesn't need to know about provider quirks.
- When a provider changes their API, only one file needs updating.

**Where:** `src/llm/providers/gpt_openai_client.py`

---

## 🏗️ E2B Sandboxed Code Execution

**What:** Python code execution uses E2B (a cloud sandbox), not local execution.

**Why:**
- **Security:** The LLM might generate malicious code. Running it locally would be dangerous.
- **Reproducibility:** The sandbox has a consistent, pre-configured environment.
- **Pre-installed packages:** Dozens of data science packages are available without installation.
- **Isolation:** Code execution can't affect the host system or other tasks.

**Where:** `src/tool/mcp_servers/python_server.py`

---

## 🏭 Factory Pattern for LLM Clients

**What:** `LLMClient()` is a factory function, not a class. It dynamically imports and instantiates the correct provider based on config.

**Why:**
- The orchestrator doesn't need to know about specific providers — it just calls `LLMClient(config)`.
- Adding a new provider requires zero changes to the orchestrator or pipeline.
- The `provider_class` name is validated before `importlib` to prevent code injection.

**Where:** `src/llm/client.py`

---

## ⚙️ Hydra for Configuration

**What:** All settings are in YAML files managed by Meta's Hydra framework.

**Why:**
- **Reproducibility:** Every experiment is fully described by its config file.
- **No code changes:** Switching models, tools, or benchmarks is just a YAML edit.
- **Composition:** Mix-and-match agent + benchmark + tool configs without duplication.
- **CLI overrides:** Tweak any setting on the fly from the command line.

**Where:** `config/` directory

---

## 🔥 Python Fire for CLI

**What:** Python Fire auto-generates the CLI from a dictionary of functions.

**Why:**
- Near-zero boilerplate — `main.py` is just ~40 lines.
- Adding a new command is one line of code.
- Automatic help generation and type conversion.
- Keeps the CLI layer thin, with all logic in importable modules.

**Where:** `main.py`

---

**Next:** [16 — API Reference](16-api-reference.md) · **Previous:** [14 — Input/Output Processing](14-input-output.md)
