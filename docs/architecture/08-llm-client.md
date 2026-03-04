# 08 — LLM Client Abstraction

> **In plain English:** MiroFlow can talk to many different AI providers (OpenAI, Anthropic, DeepSeek, etc.). This page explains how MiroFlow hides the differences between providers behind a common interface, so the rest of the code doesn't need to care which LLM you're using.

---

## The Factory Pattern

**File:** `src/llm/client.py`

When MiroFlow needs an LLM client, it doesn't create one directly. Instead, it calls a **factory function**:

```python
def LLMClient(task_id, cfg=None, llm_config=None, task_log=None, **kwargs):
    # 1. Read provider_class from config (e.g., "ClaudeOpenRouterClient")
    # 2. Dynamically import it from src.llm.providers
    # 3. Create an instance and return it
```

**Think of it as:** A vending machine. You put in a config that says "I want Claude via OpenRouter" and it gives you back a ready-to-use client. You don't need to know how the client was built internally.

> **Design rationale — Why a factory function?**
> 1. **Decoupling:** The orchestrator just calls `LLMClient(config)`. It never imports or references a specific provider class. This means adding a new provider requires zero changes to the orchestrator.
> 2. **Config-driven:** Which provider to use is a YAML setting, not a code decision. Experimenters can swap providers without touching Python.
> 3. **Security:** The factory validates that `provider_class` is a valid Python identifier before using `importlib` to import it, preventing code injection.

---

## The Base Class

**File:** `src/llm/provider_client_base.py`

All LLM providers inherit from `LLMProviderClientBase` — a Python `@dataclass` (not Pydantic) that defines the contract every provider must fulfill:

| Abstract Method | What it does |
|----------------|-------------|
| `_create_client(config)` | Set up the underlying API client (e.g., `AsyncOpenAI(...)`) |
| `_create_message(system_prompt, messages, tools, ...)` | Make the actual API call to the LLM |
| `process_llm_response(response, history, agent_type)` | Parse the response into text + a "should we stop?" flag |
| `extract_tool_calls_info(response, text)` | Pull out any tool calls from the LLM's response |
| `update_message_history(history, tool_results, exceeded)` | Add tool results to the conversation |
| `handle_max_turns_reached_summary_prompt(history, prompt)` | Special handling when asking for a final summary |

**Shared functionality** (implemented in the base class):
- `create_message()` — The unified entry point that filters history, calls the provider, and tracks token usage.
- `convert_tool_definition_to_tool_call()` — Converts MCP tool definitions to OpenAI function-calling format.
- `_remove_tool_result_from_messages()` — Removes old tool results to manage context length.
- `get_usage_log()` — Returns a formatted string of cumulative token usage.

> **Design rationale — Abstract base class:**
> The base class ensures every provider follows the same interface. The orchestrator can call `client.create_message(...)` on *any* provider and get back a consistent result. This is the [Strategy Pattern](https://en.wikipedia.org/wiki/Strategy_pattern) — the algorithm (orchestrator loop) stays the same, but the strategy (LLM provider) is swappable.

---

## Available Providers

| Class | File | API Used | Notes |
|-------|------|----------|-------|
| `ClaudeOpenRouterClient` | `claude_openrouter_client.py` | OpenRouter → Anthropic Claude | XML tool-call parsing, tiktoken for token counting |
| `ClaudeAnthropicClient` | `claude_anthropic_client.py` | Anthropic API directly | Direct API access (no middleman) |
| `GPTOpenAIClient` | `gpt_openai_client.py` | OpenAI API | Handles GPT-4o, o1, o3, o4-mini, GPT-4.1 |
| `GPT5OpenAIClient` | `gpt5_openai_client.py` | OpenAI API | GPT-5 specific handling |
| `DeepSeekOpenRouterClient` | `deepseek_openrouter_client.py` | OpenRouter → DeepSeek | |
| `MiroThinkerSGLangClient` | `mirothinker_sglang_client.py` | SGLang server | For local open-source model deployment |

### Provider-Specific Behaviors

- **OpenAI reasoning models** (`o1`, `o3`, `o4`, `gpt-5`) use `developer` role instead of `system` role — the provider client handles this transparently.
- **All clients** use **retry decorators** (`tenacity`) with exponential backoff — if an API call fails (rate limit, timeout), it automatically retries.
- **Async clients** use `AsyncOpenAI` while sync clients use `OpenAI`.

> **Design rationale — Provider-specific quirks in provider files:**
> Each provider has its own quirks (different roles, different error types, different auth). By isolating these in individual provider files, the quirks don't leak into the shared orchestrator logic. When OpenAI changes their API, you only need to update one file.

---

## Token Usage Tracking

Every provider tracks cumulative token usage per agent session:

```python
total_input_tokens: int
total_input_cached_tokens: int
total_output_tokens: int
total_output_reasoning_tokens: int
```

- Reset at the start of each session (`reset_usage_stats()`).
- Logged at the end via `get_usage_log()`.

> **Design rationale — Per-session tracking:** Knowing how many tokens each task consumed is essential for cost optimization. If a benchmark run costs $50, you can look at per-task usage to identify which tasks are expensive and why. Cached token tracking helps measure how effective caching is.

---

**Next:** [09 — Tool System](09-tool-system.md) · **Previous:** [07 — Pipeline Layer](07-pipeline.md)
