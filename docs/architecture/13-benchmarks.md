# 13 — Benchmark Evaluation System

> **In plain English:** MiroFlow isn't just an agent — it's also a benchmarking platform. You can run it against standard AI benchmarks to measure how well it performs. This page explains how that works.

**File:** `common_benchmark.py`

---

## How Benchmarking Works

The benchmark system follows a simple pipeline:

1. **Load benchmark config** — picks the dataset and evaluation criteria.
2. **Load tasks** — from HuggingFace datasets, local files, or custom loaders.
3. **Run tasks concurrently** — uses `asyncio` with configurable concurrency (multiple tasks at once).
4. **Evaluate results** — compares the agent's answers to ground truth using dataset-specific evaluation functions.
5. **Save results** — JSON logs, score summaries, per-task traces.

> **Design rationale — Built-in benchmarking:** Instead of requiring a separate evaluation harness, MiroFlow includes benchmarking as a first-class feature. This ensures that performance measurements are always consistent with how the agent actually runs, eliminating the "works differently in eval" problem.

---

## Supported Benchmarks

| Benchmark | Config File | What it tests |
|-----------|------------|---------------|
| **GAIA Validation** | `gaia-validation.yaml` | General AI assistant tasks (files, reasoning, search) |
| **GAIA Test** | `gaia-test.yaml` | GAIA test split (for leaderboard submission) |
| **HLE** | `hle.yaml` | Human-Level Evaluation (hard reasoning) |
| **HLE Text-Only** | `hle-text-only.yaml` | HLE without images or files |
| **BrowseComp EN** | `browsecomp-en.yaml` | Web browsing comprehension (English) |
| **BrowseComp ZH** | `browsecomp-zh.yaml` | Web browsing comprehension (Chinese) |
| **FutureX** | `futurex.yaml` | Future event prediction |
| **xBench DeepSearch** | `xbench-ds.yaml` | Cross-lingual deep search |
| **FinSearchComp** | `finsearchcomp.yaml` | Financial search comprehension |
| **WebWalkerQA** | `webwalkerqa.yaml` | Web navigation QA |

> **Design rationale — One config per benchmark:** Each benchmark has its own YAML file defining data paths, evaluation logic, and any special settings. This keeps benchmark definitions separate from agent configs. You can run the *same* agent against *different* benchmarks by just changing the config.

---

## Running a Benchmark

```bash
# Run GAIA validation benchmark with Claude 3.7 Sonnet
uv run main.py common-benchmark --config_file_name=agent_gaia-validation_claude37sonnet
```

This will:
1. Download the GAIA validation dataset (if not already cached).
2. Run the agent on every task in the dataset (with concurrent execution).
3. Evaluate each answer against the ground truth.
4. Save a summary JSON with scores and per-task results.

---

## Concurrency

The benchmark runner uses `asyncio` to run multiple tasks at the same time. The concurrency level is configurable — you can run 1 task at a time (for debugging) or 10+ (for speed).

> **Design rationale — Async concurrency:** Benchmark runs can involve hundreds of tasks, each taking several minutes. Running them one at a time would take days. Async concurrency lets you run many tasks in parallel, dramatically reducing evaluation time. The concurrency limit prevents overwhelming API rate limits.

---

**Next:** [14 — Input/Output Processing](14-input-output.md) · **Previous:** [12 — Logging & Observability](12-logging.md)
