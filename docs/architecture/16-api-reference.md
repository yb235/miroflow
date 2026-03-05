# 16 — API Reference Summary

> **In plain English:** This is a quick-reference table for finding specific classes and functions in the codebase. Use it as a lookup when you're reading the code or writing new features.

---

## Core Classes

| Class / Function | Location | What it does |
|-----------------|----------|-------------|
| `Orchestrator` | `src/core/orchestrator.py` | Runs the main agent loop and sub-agent loops |
| `execute_task_pipeline()` | `src/core/pipeline.py` | Executes a single task from start to finish |
| `create_pipeline_components()` | `src/core/pipeline.py` | Creates ToolManagers and OutputFormatter from config |
| `LLMClient()` | `src/llm/client.py` | Factory function — creates the right LLM provider |
| `LLMProviderClientBase` | `src/llm/provider_client_base.py` | Abstract base class for all LLM providers |
| `ToolManager` | `src/tool/manager.py` | Connects to MCP servers and executes tools |
| `TaskTracer` | `src/logging/task_tracer.py` | Pydantic model for structured task logs |
| `OutputFormatter` | `src/utils/io_utils.py` | Formats tool results and extracts `\boxed{}` answers |
| `BaseAgentPrompt` | `config/agent_prompts/base_agent_prompt.py` | Abstract base class for all prompt generators |

---

## Key Functions

| Function | Location | What it does |
|----------|----------|-------------|
| `process_input()` | `src/utils/io_utils.py` | Detects file types and creates the initial user message |
| `create_mcp_server_parameters()` | `src/utils/tool_utils.py` | Converts YAML config to MCP `StdioServerParameters` |
| `expose_sub_agents_as_tools()` | `src/utils/tool_utils.py` | Makes sub-agents appear as callable tools |
| `extract_hints()` | `src/utils/summary_utils.py` | Pre-analyzes questions for pitfalls |
| `extract_gaia_final_answer()` | `src/utils/summary_utils.py` | Post-processes and cleans up the final answer |
| `bootstrap_logger()` | `src/logging/logger.py` | Sets up the Rich console logger |
| `setup_mcp_logging()` | `src/logging/logger.py` | Configures ZMQ logging for MCP server processes |

---

## Provider Classes

| Class | File | API |
|-------|------|-----|
| `ClaudeOpenRouterClient` | `src/llm/providers/claude_openrouter_client.py` | OpenRouter → Claude |
| `ClaudeAnthropicClient` | `src/llm/providers/claude_anthropic_client.py` | Anthropic direct |
| `GPTOpenAIClient` | `src/llm/providers/gpt_openai_client.py` | OpenAI (GPT-4o, o1, etc.) |
| `GPT5OpenAIClient` | `src/llm/providers/gpt5_openai_client.py` | OpenAI (GPT-5) |
| `DeepSeekOpenRouterClient` | `src/llm/providers/deepseek_openrouter_client.py` | OpenRouter → DeepSeek |
| `MiroThinkerSGLangClient` | `src/llm/providers/mirothinker_sglang_client.py` | SGLang (local) |

---

## Prompt Classes

| Class | File | Used by |
|-------|------|---------|
| `BaseAgentPrompt` | `config/agent_prompts/base_agent_prompt.py` | (abstract) |
| `MainAgentPrompt_GAIA` | `config/agent_prompts/main_agent_prompt_gaia.py` | Main agent (GAIA benchmark) |
| `MainAgentPromptBoxedAnswer` | `config/agent_prompts/main_boxed_answer.py` | Main agent (general) |
| `MainAgentPrompt_DeepSeek` | `config/agent_prompts/main_agent_prompt_deepseek.py` | Main agent (DeepSeek) |
| `SubAgentWorkerPrompt` | `config/agent_prompts/sub_worker.py` | Sub-agents |

---

**Next:** [17 — Dependencies](17-dependencies.md) · **Previous:** [15 — Design Decisions](15-design-decisions.md)
