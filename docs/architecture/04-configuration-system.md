# 04 — Configuration System

> **In plain English:** Instead of hard-coding settings in Python, MiroFlow stores all its settings in YAML files. This means you can change how the agent behaves — which LLM to use, which tools to enable, how many turns to allow — just by editing a text file. No code changes needed.

---

## The Big Picture

MiroFlow uses **[Hydra](https://hydra.cc/)** — a configuration framework by Meta — to manage all settings. Hydra lets you:

- Write settings in human-readable YAML files.
- Compose multiple YAML files together (like building blocks).
- Override any setting from the command line.
- Use environment variables (like API keys) directly in YAML via `${oc.env:VAR_NAME}`.

> **Design rationale — Why Hydra + YAML?**
> 1. **Reproducibility:** Every experiment can be fully described by its YAML config file. Just share the YAML and anyone can reproduce your run.
> 2. **No code changes for experiments:** Want to try Claude instead of GPT? Change one line in YAML.
> 3. **Composition:** You can mix-and-match benchmark configs with agent configs without duplication.
> 4. **CLI overrides:** You can tweak any setting on the fly: `uv run main.py trace main_agent.llm.temperature=0.5`.

---

## Configuration Hierarchy

```
config/
├── agent_*.yaml           ← Top-level: "I want this agent setup"
│   defaults:
│     - benchmark: gaia-validation   ← Pulls in benchmark/gaia-validation.yaml
│     - _self_
│
├── benchmark/*.yaml       ← Dataset-specific settings (name, data paths, eval logic)
└── tool/*.yaml            ← Tool server configs (one per tool)
```

**How it works:**
1. You pick an `agent_*.yaml` file (e.g., `agent_gaia-validation_claude37sonnet.yaml`).
2. That file says "I also need the `gaia-validation` benchmark config."
3. Hydra merges everything into one big configuration object.

---

## Agent Configuration — What Goes in an `agent_*.yaml`?

Here's an annotated example:

```yaml
# === MAIN AGENT SETTINGS ===
main_agent:
  prompt_class: "MainAgentPrompt_GAIA"        # Which Python class generates the system prompt
  llm:
    provider_class: "ClaudeOpenRouterClient"   # Which LLM provider to use
    model_name: "anthropic/claude-3.7-sonnet"  # The specific model
    async_client: true                         # Use async API calls (faster)
    temperature: 0.3                           # Lower = more focused/deterministic
    top_p: 0.95                                # Nucleus sampling parameter
    max_tokens: 32000                          # Max tokens per LLM response
    openrouter_api_key: "${oc.env:OPENROUTER_API_KEY}"  # Read from .env file
  tool_config:
    - tool-reasoning                           # Which tools the main agent can use
  max_turns: -1                                # -1 = no limit (run until done)
  max_tool_calls_per_turn: 10                  # Safety limit per turn
  input_process:
    hint_generation: true                      # Pre-analyze the question for pitfalls
  output_process:
    final_answer_extraction: true              # Post-process to extract clean answer
  keep_tool_result: -1                         # -1 = keep all tool results in context
  chinese_context: false                       # Enable Chinese language handling
  add_message_id: true                         # Prevent cross-conversation cache hits

# === SUB-AGENT SETTINGS ===
sub_agents:
  agent-worker:                                # Name MUST start with "agent-"
    prompt_class: SubAgentWorkerPrompt
    llm: { ... }                               # Can use a different LLM than the main agent
    tool_config:
      - tool-searching                         # Sub-agent can search the web
      - tool-image-video                       # Sub-agent can analyze images
      - tool-reading                           # Sub-agent can read files
      - tool-code                              # Sub-agent can run Python code
      - tool-audio                             # Sub-agent can transcribe audio
    max_turns: -1
    max_tool_calls_per_turn: 10
```

> **Design rationale — Separate main/sub-agent configs:** The main agent focuses on high-level reasoning and strategy, while sub-agents handle specific information-gathering tasks. By configuring them separately, you can give the main agent a powerful (expensive) model for reasoning, and sub-agents a faster (cheaper) model for routine tasks. This saves cost without sacrificing quality.

---

## Tool Configuration — What Goes in a `tool/*.yaml`?

Each tool YAML maps to an MCP server (a separate process that provides tools):

```yaml
name: "tool-searching"                          # How the agent refers to this tool
tool_command: "python"                           # What program to run
args:
  - "-m"
  - "src.tool.mcp_servers.searching_mcp_server"  # The Python module to launch
env:
  SERPER_API_KEY: "${oc.env:SERPER_API_KEY}"     # API keys passed as env vars
  JINA_API_KEY: "${oc.env:JINA_API_KEY}"
```

> **Design rationale — One YAML per tool:** This makes tools modular and composable. You can give Agent A access to `tool-searching` and `tool-reading`, while Agent B only gets `tool-reasoning`. Each tool is independently configured with its own API keys and settings.

---

## Environment Variables

Secrets like API keys are stored in a `.env` file (never committed to git) and referenced in YAML using Hydra's `${oc.env:VAR_NAME}` syntax:

```
OPENROUTER_API_KEY=sk-or-v1-...
SERPER_API_KEY=...
JINA_API_KEY=...
E2B_API_KEY=...
OPENAI_API_KEY=...
```

> **Design rationale — Env vars for secrets:** This follows the [twelve-factor app](https://12factor.net/config) principle: secrets belong in the environment, not in code or config files. The `.env.template` file shows you what keys are needed without revealing actual values.

---

**Next:** [05 — Agent Architecture](05-agent-architecture.md) · **Previous:** [03 — Entry Points & CLI](03-entry-points-and-cli.md)
