# exercise-05: Inspect Agent Eval End-to-End

**Estimated effort:** 4 hours

## Objective

Wire a custom tool-use eval end-to-end in **Inspect** — dataset, agent solver with tools, sandbox setup, custom trajectory scorer that emits the Chapter 2 signal vector, and a report. The eval task itself is small and self-contained (a synthetic task family you own), but the scaffolding is production-shape: it can be re-run, re-scored without re-inferring, and shared as an eval-log bundle for a reviewer to reproduce. The deliverable is the *harness*, not any particular model number. This is the template you will re-use for internal agent evals after the course.

## Prerequisites

- mod-108 Chapter 6 (Inspect's agent harness end-to-end).
- mod-104 Chapter 5 (Inspect solver / scorer basics).
- Exercise 01's trajectory scorer (you will port its per-step signals into Inspect scorer metadata).
- Python 3.11+ with `inspect_ai` (`pip install inspect-ai`). Access to at least one hosted model (Anthropic, OpenAI, or a self-hosted OpenAI-compatible endpoint).
- Optional: Docker, if you pick a task family that requires process isolation (recommended stretch).

## The task family

Pick **one** self-contained tool-using task family. Two starter options; propose your own if neither fits.

### Option A — the KV-lookup task (Chapter 6's skeleton)

Extend the Chapter 6 example. 40+ items, each with a per-item seed KV store, at least three distinct question shapes:

- Direct lookup (`"What is the capital of X?"` → single `kv_get`).
- Two-hop lookup (`"What is the population of the capital of X?"` → two `kv_get` calls).
- Enumeration (`"List all countries whose population exceeds 10 million."` → `kv_list_keys` + multiple `kv_get`).

The multi-shape design forces the agent to plan; a single-shape family would be pass@1 = 1.0 for any capable model and would tell you nothing about the harness.

### Option B — a synthetic file-search task

40+ items over a per-item mock filesystem (implemented as a dict of `path → content`). Tools: `read_file(path)`, `list_dir(path)`, `grep(pattern, path)`, `submit(answer)`. Questions like "find the file that defines `class UserProfile` and return its path" or "which function in `src/utils.py` handles email validation?"

## Requirements

### Part A — the dataset

Ship `data/<task>.jsonl` with 40+ items. Each row has `input`, `target`, and a `metadata` blob with the per-item seed (the KV dict for Option A, the filesystem dict for Option B) plus an `item_id` and a difficulty tag (`easy` / `medium` / `hard` — used later for per-slice reporting).

The seed is the *task's* state, not the *agent's* state. It is materialized into the sandbox at the start of every sample and reset between samples.

### Part B — the tools

Implement tools as `@tool`-decorated async callables. Every tool must:

- Read from a per-sample store (`inspect_ai.util.store()`), not module-level state.
- Return typed, well-formed strings (JSON when returning structured data; a clear error message with an "ERROR:" prefix when returning failure).
- Document its schema in the docstring (Inspect derives the JSON schema from signature + docstring).
- Have at least one call path that returns an *error* (the model should have to handle both success and error responses).

### Part C — the agent solver

Compose the solver chain:

1. **A seeding solver** that reads `metadata` and populates the per-sample store.
2. **A `basic_agent`** with your tools, a documented system prompt (kept in a file, not inline), and a `max_messages` budget you'll pin in the manifest.

Do not use hard-coded model IDs in the task; the harness's `--model` flag picks the model at run time.

### Part D — the trajectory scorer

Ship a `@scorer` that returns a `Score` where:

- `value` = final-answer correctness (0.0 / 1.0). Correctness check is exact-match after a documented normalization; log the raw final answer for auditability.
- `metadata` includes the exercise-01 signal vector:
  - `tool_calls` — total across the trajectory.
  - `validity_rate` — fraction of tool calls with parseable arguments.
  - `execution_success_rate` — fraction of tool results without `ERROR:`.
  - `redundant_call_rate` — fraction of tool calls repeating a `(name, canonicalized args)` earlier in the same trajectory.
  - `termination` — one of `submitted`, `budget_exhausted`, `harness_error`.
  - `final_answer` (raw string).

Register at least four custom `@metric` aggregators (`validity_rate`, `execution_success_rate`, `redundant_call_rate`, `mean_tool_calls`) so they show up in the Inspect report alongside `accuracy`.

### Part E — run

Run the eval end-to-end against at least two models. Example:

```bash
inspect eval my_task.py@my_task \
  --model anthropic/claude-3-5-sonnet-latest \
  --limit 40 --epochs 2 \
  --log-dir logs/my_task/sonnet

inspect eval my_task.py@my_task \
  --model openai/gpt-4o-mini \
  --limit 40 --epochs 2 \
  --log-dir logs/my_task/4o-mini
```

Record wall-clock, total tokens, and total cost per run. Log Inspect's per-sample usage into the log directory (default behavior).

### Part F — re-score without re-inferring

Modify the redundancy definition (or add a new metric — e.g. a milestone check from Chapter 5 Shape 1) and re-run the scorer against the existing eval logs *without* re-inferring:

```bash
inspect score logs/my_task/sonnet --scorer my_scorer.py@my_task_scorer_v2
```

Report the two versions of the score side-by-side. The point: demonstrate that the eval-log pattern lets you iterate on scorers cheaply. Include the wall-clock comparison in the report.

### Part G — the report

Ship `REPORT.md` (1–2 pages):

- Task family, dataset size, difficulty tag distribution.
- Per-model headline: accuracy, per-step signal means, termination distribution, cost.
- Per-slice breakdown (by difficulty tag).
- Model comparison: a two-row Pareto snippet ($ per task vs accuracy) plus a brief interpretation.
- Reproducibility manifest (see below).
- A one-paragraph "re-scoring" note demonstrating Part F.

### Part H — the manifest

Ship `MANIFEST.md` with:

- Inspect version (`inspect_ai.__version__`).
- Models used (provider, ID, version string, temperature, top-p).
- System prompt file SHA.
- Tool definitions file SHA.
- Budget (`max_messages`, per-request `max_tokens`).
- Dataset SHA.
- Run identifier, `.eval` log file paths, run start/end timestamps.
- Headline numbers.

The `.eval` log files themselves are the reproducibility artefact — ship them (or a shareable subset) in the bundle.

## Starter guidance

- **Start with 5 items and one model.** Iterate on tools, prompt, and scorer against 5 items until the whole loop works end-to-end. Then scale to 40 items × 2 models × 2 epochs.
- **Design the tools to make the model earn it.** If `kv_get` always returns the answer verbatim on the first call, the trajectory will always be one step and you'll learn nothing. Include lookup misdirection (keys that require enumeration first), typos-in-keys, and near-miss keys that force the agent to read carefully.
- **Log everything to `Score.metadata`.** Anything you might want to slice or plot later belongs there. Once the eval log is written, you can re-analyze without re-running the model. The Chapter 6 rule of thumb: log the vector even when the current report only uses the accuracy.
- **Do not fight Inspect's dataset schema.** If your task needs richer per-sample state than `input`, `target`, and a `metadata` dict, put it in `metadata` as a serializable blob and let the seeding solver materialize it. Writing a custom `Dataset` subclass is premature for this exercise.
- **Docker sandbox is optional here.** If your tools are pure Python against an in-memory store, `sandbox=None` is correct. Only reach for `sandbox="docker"` if a tool actually needs process isolation. If you do use Docker, ship the `Dockerfile` and `compose.yaml`.
- **Use `--epochs 2` at minimum.** One epoch per sample gives you a point estimate; two epochs give you the beginning of a variance estimate. `pass^k` reporting starts to make sense at `k ≥ 3`.
- **Use `inspect view` before writing the report.** Click through 10 trajectories in the UI. If any of them look "wrong" (looping, malformed tools, obvious prompt issues), fix and re-run before publishing a number.

## Acceptance criteria

- The eval runs end-to-end: `inspect eval my_task.py@my_task --model <M> --limit 40` produces a complete `.eval` log without errors.
- At least four custom metrics (validity, execution success, redundancy, mean tool calls) appear in the Inspect report alongside `accuracy`.
- The scorer's `Score.metadata` records the Chapter 2 signal vector per sample; the report cites specific values.
- Two models were run under the same task + budget; the report includes a per-model comparison.
- `inspect score <log> --scorer <new>` was demonstrated to re-score an existing log with a modified scorer without re-inferring; the wall-clock delta is reported.
- The manifest pins Inspect version, models, prompts, tool file, dataset SHA, and budget. The `.eval` log files are shipped.

## Stretch goals

- **Sandboxed variant.** Extend one tool to shell out (`bash` running actual commands against a per-sample tempdir), switch `sandbox="docker"`, ship a Compose file, and rerun. Report the wall-clock overhead vs the pure-Python variant.
- **Milestone scorer.** Add a Chapter-5-Shape-1 milestone scorer to the task. Chain both scorers (`scorer=[trajectory_scorer(), milestone_scorer()]`); report both.
- **Judge-graded partial credit.** Add a Chapter-5-Shape-2 rubric scorer using a judge model. Audit its agreement with hand-graded partial-credit on 10 items using the exercise-04 protocol.
- **`pass^k` reporting.** Extend the aggregator to compute `pass^k` from a `k=3` epochs run and report both `pass@1` and `pass^3` per model.
- **Cross-provider adapter.** Run the same task against a self-hosted OSS model via a vLLM OpenAI-compatible endpoint. Report gaps against the hosted models. Non-comparable numbers are still informative — a self-hosted model that scores much lower is likely losing to prompt-template mismatch, not raw capability, and the cross-provider run is the fastest way to expose it.
- **Public export.** Anonymize a subset of the eval log (redact API keys, per-sample content that includes proprietary text) and write a short "reviewer-quickstart" README. Ship the bundle as if to a colleague who's never seen the task; test that a fresh clone can `inspect view logs/anon-bundle` and browse trajectories.
