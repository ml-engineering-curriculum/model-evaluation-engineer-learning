# exercise-02: Multi-Runner Orchestration

**Estimated effort:** 4 hours

## Objective

Build the multi-runner orchestration plane from Chapter 3: a small platform service that accepts eval-run requests by (task, model, config), resolves registry references, picks a runner via a documented dispatch policy, invokes the runner through a common adapter contract, and normalizes results into a per-item schema. The deliverable is a working plane with **at least two** runner adapters (lm-evaluation-harness plus one of Inspect or a minimal internal harness), a **model-client layer** that enforces version-pinning across two model backends, and a demonstration that the same task run against the same model on two different adapters resolves through the plane consistently — or is refused if the plane cannot determine equivalence.

## Prerequisites

- mod-111 Chapter 3 (multi-runner orchestration) and Chapter 2 (registry) — the exercise builds on the registry from exercise-01, or a stub if you're starting fresh.
- mod-104 (LLM benchmark harnesses) for enough familiarity with lm-evaluation-harness or Inspect to write an adapter without reading the harness's source in-depth for the first time.
- Python 3.11+ with `fastapi` (or `flask`), `uvicorn`, a task queue (`rq`, `celery`, or a small threading-based queue for the exercise), and access to at least one hosted model (a hosted-vendor API key) plus one alternative (a local server, a second vendor, or a mock).

## Requirements

### Part A — the plane's user-facing API

Ship a small HTTP service with at minimum:

- `POST /v1/eval-runs` — accepts a request body with `task`, `model`, and `config` (`seed`, `sample_size`, `priority`, `budget_tokens`, `budget_wallclock_min`, `attributions`). Returns a `run_id`. Rejects requests whose registry references cannot be resolved, whose model reference is unversioned, or whose `budget_tokens` is smaller than the historical projection for that (task, model) pair by more than a documented tolerance.
- `GET /v1/eval-runs/{run_id}` — returns run status, resolved artifact hashes, dispatched runner, dispatched runner version, and (once complete) a link to results.
- `POST /v1/eval-runs/{run_id}/cancel` — cancels a queued or running run; sends a stop signal to the adapter.
- `GET /v1/eval-runs/{run_id}/events` — streams the plane's normalized event log for the run.

The API is deliberately small. Every field a caller sets is one the plane will validate; the plane rejects requests it cannot make sense of, rather than silently interpreting.

### Part B — the adapter contract

Implement the `RunnerAdapter` protocol from Chapter 3 as a Python `Protocol` (or ABC) and ship at least two implementations.

Required adapters (implement at least two of):

- **`lm_eval_harness_adapter.py`** — spawns `lm-evaluation-harness` as a subprocess (or uses its Python API), materializes a task YAML from the resolved artifacts, and translates the JSON output into per-item scores.
- **`inspect_adapter.py`** — writes an `@task`-decorated Python file (or uses `inspect_ai`'s in-process API), invokes `inspect eval`, and translates the resulting logs into per-item scores.
- **`internal_harness_adapter.py`** — a minimal in-house adapter that runs a specific task shape (e.g., HarmBench-style refusal grading) directly. Useful when you want the exercise to run without external harness dependencies.

Each adapter conforms to `RunnerAdapter` and does the following:

- Registers its `name` and `supported_task_shapes` at import time.
- Implements `can_run(resolved_task) -> bool` correctly for the task shapes it registers.
- Implements `prepare(...)` to materialize the runner's native config from platform-resolved artifacts. **Refuses** to run if any component of the request diverges from the registered task's declared configuration.
- Implements `execute(...)` to run the runner, emit events to the plane's `event_sink` (`run.started`, `item.completed`, `error`, `budget.consumed`), and respect the platform-provided `BudgetHandle` (the adapter calls `budget.charge(tokens=..., wallclock_s=...)` and stops if `budget.exhausted()`).
- Implements `parse_results(...)` to emit per-item results conforming to the platform's `PerItemScore` schema.
- Implements `lineage_snapshot(...)` to capture the runner version, pinned dependency snapshot (an `image_digest` for a container-image adapter, a `pip freeze` hash otherwise), and the environment fingerprint.

### Part C — the model-client layer

Ship a thin `model_client/` package that sits between adapters and model backends:

- A single `ModelClient` interface with `call(messages, decoding_config, stop_sequences, seed=None, cache_hints=None) -> ModelResponse`.
- Concrete backends for at least two of: `AnthropicBackend`, `OpenAIBackend`, `LiteLLMBackend`, `VLLMBackend`. Each backend translates the common request into the backend's native shape.
- **Version-pinning enforcement.** The client rejects `provider:model` routes without a version qualifier at construction time. `anthropic:claude-sonnet-4-6@2026-02-01` is accepted; bare `anthropic:claude-sonnet-4-6` is not, unless the platform's `ALIASES` config explicitly maps the alias to a versioned target.
- **Shared retry / backoff.** Adapters do not implement retries. The client handles 429s and 5xxs with jittered exponential backoff (starting ~500ms, capped at ~30s, max 6 attempts). Every retry emits an event that the plane's event sink can attribute to the run.
- **Usage reporting.** Every response includes `usage.prompt_tokens`, `usage.completion_tokens`, and (when the backend supports it) `usage.cached_prompt_tokens`. The client raises `UsageParseError` if the backend response is missing usage fields it should have.

### Part D — the dispatcher

Ship `dispatcher.py`:

- On receipt of a request, resolves the task through the registry (uses the exercise-01 registry or a stub).
- Consults the task's registered `runner_compatibility` list. If empty, refuses the request. If a singleton, dispatches to that runner. If multiple, applies a policy: pick the runner with the lowest current backlog and the lowest recent error rate; break ties by the task's registered preferred runner.
- Records the dispatch decision (which runner, why) in the `run_event_log`. A run that could have gone to two runners has an event that says so and names the choice.
- Refuses to switch runners between retries of the same run — a run that started on lm-eval-harness and failed does not silently retry on Inspect.

### Part E — the consistency demonstration

Write a `CONSISTENCY.md` that demonstrates two things end-to-end:

1. **Cross-adapter execution of a task with a single registered runner.** Register a task whose `runner_compatibility` is `[lm-eval-harness]`. Submit two runs against different models. Show that both run through the plane, land in the warehouse (a JSONL file is fine for the exercise), and produce comparable per-item results.
2. **Cross-runner naming for the "same" task run under different runners.** Register two distinct tasks (`harmbench_standard.inspect@rev` and `harmbench_standard.lm_eval_harness@rev`) that share the underlying dataset but use different runners. Submit runs against the same model on each. Show that the plane does **not** join them silently — the two aggregate metrics are labeled with different task hashes and any cross-adapter comparison in the warehouse must be an explicit join, not an accidental one.

## Starter guidance

- **Start with the adapter contract, not with any runner.** A working `Protocol` with a passing type check and a `MockRunnerAdapter` that returns fixed results is the scaffolding on which the real adapters can land incrementally.
- **The plane's event schema is the shared vocabulary.** Every adapter's events go through the same schema; the plane's tests can then be adapter-agnostic (submit a request, assert the events arrive in the expected sequence).
- **Model-client version enforcement is a five-line thing that pays for itself.** Do it early; every backend you write inherits the property.
- **Budget handles are opaque to the adapter.** The adapter charges the handle and reads `budget.exhausted()`; the plane owns the accounting.
- **A small fixture task pair is enough for the demonstration.** You do not need to reproduce a full leaderboard; a 20-item HarmBench slice run on two models is a legible demonstration.
- **Instrument the plane with a structured logger from day one.** The event stream is easier to inspect than a debugger for the async dispatch path.

## Acceptance criteria

- At least two runner adapters implement the `RunnerAdapter` contract and can execute an end-to-end run through the plane against a real (or stubbed) model backend.
- The model-client layer rejects unversioned model routes at construction time and handles 429 / 5xx with jittered exponential backoff. Retries are attributed to the run's budget.
- The dispatcher picks a runner via the documented policy and records the choice in the event log; runs do not silently switch runners between retries.
- Every completed run has a resolved artifact-hash bundle (task, dataset, prompt, judge if present, model) and a `runner_version` field in its lineage snapshot.
- The plane's per-item results conform to the platform's canonical schema and can be re-aggregated to produce the runner-reported aggregate; a persistent mismatch between runner-reported and re-aggregated results is emitted as a lint event.
- `CONSISTENCY.md` demonstrates the two scenarios above with runnable commands and correct results.
- A run submitted with an unresolved registry reference, an unversioned model route, or a below-projection budget is rejected with a machine-readable error.
- A submitted run can be canceled mid-flight and the adapter respects the stop signal within a documented deadline.

## Stretch goals

- **A HELM adapter.** Ship a HELM adapter and demonstrate that a HELM scenario runs through the plane with the same event and result shape as the lm-eval-harness scenario, exposing the fact that "MMLU" under HELM and under lm-eval-harness are two distinct tasks in the registry.
- **In-process serving switch.** Wire the model-client layer to a local vLLM (or SGLang) server for one of the backends; demonstrate an eval that runs against the local server via the plane exactly as it would against the hosted vendor.
- **Preemption test.** Submit a research-sweep run that occupies the queue, then submit a release-blocking run and demonstrate that the plane preempts the sweep. Requires reserved concurrency slots from Chapter 4's mechanics — a soft dependency on exercise-03.
- **Adapter version pin registry.** Ship a `registry adapters` command that lists the pinned runner versions across the plane and refuses to promote an adapter's runner-version pin without a re-baseline demonstration on a stable checkpoint.
