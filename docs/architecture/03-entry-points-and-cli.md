# 03 — Entry Points & CLI

> **In plain English:** MiroFlow is a command-line tool. You type a command in your terminal, and it does the rest. This page explains what commands are available and how they work.

---

## How the CLI Works

MiroFlow uses [Python Fire](https://github.com/google/python-fire) — a library that automatically turns Python functions into CLI commands. The entry point is `main.py`.

When you run:

```bash
uv run main.py trace --config_file_name=agent_quickstart_reading --task="What is…?"
```

Here's what happens behind the scenes:

1. **Python Fire** looks up `"trace"` in a dictionary of commands.
2. It finds that `"trace"` maps to `utils.trace_single_task.main`.
3. It calls that function, passing `--config_file_name` and `--task` as arguments.

> **Design rationale — Why Python Fire?** Fire requires almost zero boilerplate. You just give it a dictionary of `{command_name: function}` and it handles argument parsing, help generation, and type conversion automatically. This keeps `main.py` clean and easy to extend — adding a new command is just one line.

---

## Available Commands

| Command | What it does | When to use it |
|---------|-------------|----------------|
| `trace` | Runs a single task with an agent | Day-to-day use, quick experiments |
| `common-benchmark` | Runs a full batch benchmark evaluation | Measuring performance at scale |
| `eval-answer` | Evaluates answers from existing log files | Re-scoring without re-running |
| `avg-score` | Calculates average benchmark scores | Aggregating results across runs |
| `score-from-log` | Computes scores from log files | Quick analysis of past runs |
| `prepare-benchmark` | Downloads and prepares benchmark datasets | First-time setup |
| `print-config` | Prints the resolved Hydra configuration | Debugging config issues |

---

## Example: Running Your First Task

```bash
# 1. Tell MiroFlow to "trace" (run) a single task
# 2. Use the "agent_quickstart_reading" config (defines which LLM + tools to use)
# 3. Give it a question and a file

uv run main.py trace \
  --config_file_name=agent_quickstart_reading \
  --task="What is the first country listed in the XLSX file that have names starting with Co?" \
  --task_file_name="data/FSI-2023-DOWNLOAD.xlsx"
```

**Expected output:** `\boxed{Congo Democratic Republic}`

---

## What Happens Inside `main.py`

```python
# Simplified version of main.py
if __name__ == "__main__":
    fire.Fire({
        "trace":            utils.trace_single_task.main,
        "common-benchmark": common_benchmark.main,
        "eval-answer":      utils.eval_answer_from_log.main,
        "avg-score":        utils.calculate_average_score.main,
        "score-from-log":   utils.calculate_score_from_log.main,
        "prepare-benchmark": utils.prepare_benchmark.main,
        "print-config":     print_config,
    })
```

That's really it — the CLI layer is intentionally thin. All the real work happens in the functions it calls.

> **Design rationale — Thin CLI layer:** By keeping `main.py` as just a routing table, the actual logic stays in focused, testable modules. If you wanted to use MiroFlow as a library (not from the command line), you'd just call those same functions directly.

---

**Next:** [04 — Configuration System](04-configuration-system.md) · **Previous:** [02 — Directory Structure](02-directory-structure.md)
