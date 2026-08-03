# Inspect's Agent Harness End-to-End

The previous five chapters covered concepts and canonical benchmarks. This chapter is the *reference implementation*: how to stand up an agent eval from scratch using the UK AISI Inspect framework — dataset, solver, tools, sandbox, custom trajectory scorer, and the reporting pass — so that the vector from Chapter 2 (final-answer correctness, per-step signals, budget rollups, termination reasons) drops out of the run naturally. Inspect is the right harness for this because agent-loop plumbing (tool dispatch, message-history bookkeeping, sandbox lifecycle, per-sample resume) is first-class in it, and the eval log format is the richest reproducibility artefact of any harness in mod-104. When you are building an *internal* agent eval — one that isn't SWE-bench or WebArena, one where the tool surface matches your product — Inspect's `basic_agent` + custom tools + custom scorer pattern is the shortest path from a task spec to a defensible number.

Everything here builds on mod-104 Chapter 5 (Inspect solver/scorer basics); if that chapter is not fresh, re-skim before continuing. This chapter goes deeper on `basic_agent`, custom tools, sandboxes, and the trajectory-level scorer glue.

## The scenario

To keep the example concrete, we'll build an eval for a synthetic tool-using task: **"answer a factual question that requires at least one lookup in a local key-value store, and return the answer as `Final: <answer>`."** The eval has 20 items, each carrying a question, a target answer, and a KV store that the sandbox will pre-populate. The agent gets two tools — `kv_get(key)` and `kv_list_keys()` — plus a `submit` action to end the trajectory.

This is not a benchmark you would publish; it is a shape that lets the code in this chapter be actually runnable in about 100 lines with no external service dependencies.

## The dataset

A JSONL file per item:

```jsonl
{"input": "What is the capital of Coastal Republic?", "target": "Newport Harbor", "metadata": {"item_id": "kv_0001", "kv_seed": {"country:coastal_republic:capital": "Newport Harbor", "country:coastal_republic:population": "12_000_000"}}}
{"input": "What is the population of Coastal Republic?", "target": "12_000_000", "metadata": {"item_id": "kv_0002", "kv_seed": {"country:coastal_republic:capital": "Newport Harbor", "country:coastal_republic:population": "12_000_000"}}}
```

Each item's `kv_seed` becomes the starting state of the sandbox for that sample; the tool implementations read from it. In production evals this seed might be a database dump or a repo snapshot; here it is a small dict.

Load with Inspect's JSON loader:

```python
from inspect_ai.dataset import json_dataset, FieldSpec

def kv_dataset():
    return json_dataset(
        "data/kv_tasks.jsonl",
        FieldSpec(input="input", target="target", metadata=["item_id", "kv_seed"]),
    )
```

## The tools

Inspect tools are `@tool`-decorated async callables. Their docstrings become the schemas the model sees (parsed from the signature and the docstring), so write them with the model reader in mind.

```python
from inspect_ai.tool import tool
from inspect_ai.util import store

@tool
def kv_get():
    async def execute(key: str) -> str:
        """Retrieve a value from the local key-value store.

        Args:
            key: The key to look up. Must be exact-match; use kv_list_keys() to browse.

        Returns:
            The stored value as a string, or "NOT_FOUND" if the key does not exist.
        """
        kv = store().get("kv_store", {})
        return kv.get(key, "NOT_FOUND")
    return execute

@tool
def kv_list_keys():
    async def execute() -> str:
        """List all keys currently in the local key-value store.

        Returns:
            A JSON-formatted list of all available keys.
        """
        import json
        kv = store().get("kv_store", {})
        return json.dumps(sorted(kv.keys()))
    return execute
```

Two Inspect-specific idioms in play:

- **`store()`** — a per-sample key-value scratch space that persists across solver / tool invocations within one sample and is reset between samples. It is the right place for stateful per-sample setup; the alternative (a module-level global) leaks across samples in a parallel run.
- **The docstring is the tool schema.** Inspect derives the OpenAPI-shaped tool schema for the model from the function signature and the docstring. A `Returns:` section is worth writing; the model uses it to plan.

## A solver that seeds the sandbox and runs the agent

The solver chain has three steps: seed the per-sample store from `metadata.kv_seed`, install the tools, and run the ReAct-style agent loop bounded by a step / token cap.

```python
from inspect_ai.solver import solver, TaskState, Generate, basic_agent
from inspect_ai.tool import Tool
from inspect_ai.util import store

@solver
def seed_kv_store():
    async def solve(state: TaskState, generate: Generate) -> TaskState:
        seed = state.metadata.get("kv_seed", {})
        store().set("kv_store", dict(seed))
        return state
    return solve

def kv_agent_solver(max_messages: int = 12):
    from inspect_ai.solver import system_message
    return [
        seed_kv_store(),
        basic_agent(
            init=system_message(
                "You are a research assistant. Answer the user's question using the "
                "provided key-value store tools. When you have the answer, output it as "
                "'Final: <answer>' and stop."
            ),
            tools=[kv_get(), kv_list_keys()],
            max_messages=max_messages,
        ),
    ]
```

`basic_agent` is the ReAct-loop primitive: it repeatedly (a) generates an assistant message, (b) if the message contains tool calls, dispatches them and appends results, (c) if the message contains no tool calls, treats it as terminal. The `max_messages` cap is the step-budget from Chapter 2 — this is the number that goes into the reproducibility manifest as `max_turns`.

## The trajectory scorer

Now the interesting part: a scorer that emits the Chapter 2 vector. Inspect scorers return a `Score` object, but a `Score` can carry structured metadata alongside its `value`. We use the metadata channel for the trajectory signals.

```python
import json
import re
from inspect_ai.scorer import (
    scorer, Score, Target, accuracy, mean, metric,
)
from inspect_ai.solver import TaskState

FINAL_RE = re.compile(r"^\s*Final:\s*(.+?)\s*$", re.MULTILINE)

def _extract_final(text: str) -> str | None:
    match = FINAL_RE.search(text or "")
    return match.group(1).strip() if match else None

def _tool_call_stats(messages) -> dict:
    total = 0
    valid = 0
    successful = 0
    seen_calls = set()
    redundant = 0

    for msg in messages:
        if getattr(msg, "role", None) == "assistant":
            for call in (msg.tool_calls or []):
                total += 1
                try:
                    args = call.arguments if isinstance(call.arguments, dict) \
                        else json.loads(call.arguments)
                    valid += 1
                    key = (call.function, json.dumps(args, sort_keys=True))
                    if key in seen_calls:
                        redundant += 1
                    seen_calls.add(key)
                except Exception:
                    pass
        elif getattr(msg, "role", None) == "tool":
            content = getattr(msg, "content", "") or ""
            if not content.lower().startswith(("error", "exception", "not_found")):
                successful += 1

    def _rate(numer: int, denom: int) -> float:
        return numer / denom if denom else 0.0

    return {
        "tool_calls": total,
        "validity_rate": _rate(valid, total),
        "execution_success_rate": _rate(successful, total),
        "redundant_call_rate": _rate(redundant, total),
    }

@metric
def validity_rate():
    def compute(scores):
        vals = [s.score.metadata.get("validity_rate", 0.0) for s in scores]
        return sum(vals) / len(vals) if vals else 0.0
    return compute

@metric
def redundant_call_rate():
    def compute(scores):
        vals = [s.score.metadata.get("redundant_call_rate", 0.0) for s in scores]
        return sum(vals) / len(vals) if vals else 0.0
    return compute

@metric
def mean_tool_calls():
    def compute(scores):
        vals = [s.score.metadata.get("tool_calls", 0) for s in scores]
        return sum(vals) / len(vals) if vals else 0.0
    return compute

@scorer(metrics=[accuracy(), validity_rate(), redundant_call_rate(), mean_tool_calls()])
def kv_trajectory_scorer():
    async def score(state: TaskState, target: Target) -> Score:
        final = _extract_final(state.output.completion)
        target_text = target.text.strip()

        correct = final is not None and final.lower() == target_text.lower()
        stats = _tool_call_stats(state.messages)
        stats.update({
            "final_answer": final,
            "termination": "submitted" if final else "budget_exhausted",
        })

        return Score(
            value=1.0 if correct else 0.0,
            answer=final,
            explanation=(
                f"target={target_text!r}, got={final!r}, "
                f"tool_calls={stats['tool_calls']}"
            ),
            metadata=stats,
        )
    return score
```

Three things to notice:

- **The scorer returns a single `Score` with structured metadata.** `value` is the final-answer correctness (the primary metric). Everything else — tool-call stats, termination reason, extracted final answer — goes in `metadata`. This preserves the Chapter 2 vector without inventing a new abstraction.
- **Custom `@metric` aggregators pull from metadata.** `validity_rate`, `redundant_call_rate`, `mean_tool_calls` are per-eval aggregates over the per-sample metadata. They show up in the eval report alongside `accuracy`.
- **The redundancy detector is deliberately simple.** A `(function_name, canonicalized_args)` key is enough for the demonstration; a production scorer might canonicalize URLs, normalize whitespace, or treat successive `list_keys()` calls as always-redundant. Keep the scorer transparent so a reader can see what "redundant" means.

## The task

```python
from inspect_ai import task, Task

@task
def kv_lookup_agent(max_messages: int = 12):
    return Task(
        dataset=kv_dataset(),
        solver=kv_agent_solver(max_messages=max_messages),
        scorer=kv_trajectory_scorer(),
        sandbox=None,        # no external sandbox needed; state lives in store()
        message_limit=max_messages,
    )
```

For a task that needed process isolation (running untrusted shell commands, applying a patch to a repo), swap `sandbox=None` for `sandbox="docker"` and ship a `Dockerfile` + `compose.yaml` alongside the task file. Inspect handles container lifecycle per sample.

## Running and reporting

```bash
inspect eval kv_agent.py@kv_lookup_agent \
  --model anthropic/claude-3-5-sonnet-latest \
  --limit 20 \
  --epochs 3 \
  --log-dir logs/kv_agent
```

Three notes on the CLI arguments:

- **`--epochs 3`** re-runs every sample 3 times. Inspect aggregates the per-metric values across epochs; with `pass^k`-style reporting you would post-process the eval log to compute the k-consecutive-successes number described in Chapter 2.
- **`--log-dir`** is where the reproducibility bundle lands. Each run produces one `.eval` file — a self-contained log of every sample's full trajectory (messages, tool calls, tool results), the scorer verdict, the aggregate metrics, and the resolved config (model, prompt, temperature, seed, tool schemas). This is the artefact you ship for reproducibility.
- **`inspect view logs/kv_agent`** opens a browser UI that lets a reviewer click through per-sample transcripts. For any trajectory-based eval, this UI is where the model's actual failure modes become legible. Any eval report you publish should be paired with a shareable eval-log bundle.

## Re-scoring without re-inferring

The Inspect trick most under-used in agent eval: `inspect score`. Given an existing eval log, run a *different* scorer against the same recorded trajectories without re-issuing inference calls. This is important for two reasons:

- **Rubric iteration is free.** If you decide to tighten the redundancy detector, add a milestone scorer, or swap in a judge-based partial-credit scorer, `inspect score old.eval --scorer new_scorer.py` runs the new scorer against the recorded trajectories in seconds. You do not re-pay the agent-inference bill.
- **Judge-drift audits are cheap.** Re-run last quarter's rubric against this quarter's judge model on the same recorded trajectories. Compare per-sample verdicts to detect drift (mod-105 Chapter 5's protocol, applied to trajectories).

The corollary: log everything. `Score.metadata` is where trajectory-level features you might want to re-analyze later live. Log the tool-call stats even if the current report does not use them; a future re-scoring pass can pull them out without re-running the agent.

## Wiring in the four Chapter 2 signals

Map back to Chapter 2's four signals and check the coverage of the code above:

- **Final-answer correctness.** `Score.value`.
- **Per-step tool-call correctness.** `metadata.validity_rate`, `metadata.execution_success_rate`, `metadata.redundant_call_rate`. Tool-choice correctness — the *fourth* per-step signal — is not automatic here; add it if you have a reference solution, or defer to a judge rubric (Chapter 5).
- **Budget consumption.** `metadata.tool_calls` plus the built-in Inspect per-sample token / latency logging (in the eval log's `usage` fields). Wall-clock is in the log per sample; dollars are computed post-hoc from `usage` and provider price tables.
- **Termination reason.** `metadata.termination`. Extend with `harness_error` and `abort_by_scorer` if your task has scorer-triggered aborts.

The vector is complete. The report script (a small notebook or `pandas` script that reads the eval log JSON) computes medians, p95s, Pareto plots, and the per-slice breakdowns. Inspect's log is JSON at rest; a 50-line pandas transform gets you the standard report.

## Scaling this to a real eval

The KV-lookup skeleton generalises to the benchmarks in Chapters 3 and 4:

- **SWE-bench.** Replace the KV tools with `bash`, `python`, `str_replace_based_edit_tool` (or the SWE-agent tool set). Replace `sandbox=None` with a Docker sandbox that mounts the per-instance repo image. Replace the target-string check with a shell-out to the SWE-bench grader (`swebench.harness.grading`) that returns pass / fail per prediction. Ship a compose file per instance-family; use `--max-workers` to control parallelism.
- **WebArena.** Replace tools with a Playwright-backed browser toolset (`click`, `type`, `navigate`, `screenshot`, `get_dom`). Sandbox is the WebArena docker-compose stack, brought up fresh per sample with the reset script from Chapter 4. Success criterion is the WebArena evaluator that inspects the browser and DB state; call it from the scorer and record its reasoning.
- **GAIA.** Replace tools with a web-search + file-read + Python-execution set. There is no sandbox reset (the internet is stateful); the manifest records the run window. Success criterion is exact-match against the target with a short-answer normalizer; ship the normalizer as a first-class part of the scorer, and log the raw model final answer so the reader can audit the normalizer.

Each of these is a bigger engineering exercise, but the *shape* — dataset, seeding solver, tool-using solver, trajectory-vector scorer, log-based reporting — is what Inspect gives you for free.

## When Inspect isn't the right harness

Two situations where you'd reach for something else:

- **Reproducing a specific SWE-bench or WebArena leaderboard number.** Use the benchmark's *reference* harness (the SWE-bench `swebench` package, the WebArena `browser_env` package). A from-scratch Inspect reimplementation will not match the published number to within the reproduction tolerance because the reference harness's prompt templates and tool descriptions are what the leaderboard was tuned against. Once you can reproduce the reference number, you can port to Inspect for internal comparison work.
- **Enterprise-internal agent frameworks.** If your product's agent is built on LangGraph, LlamaIndex, an in-house framework, or a vendor-supplied stack (OpenAI Assistants / Responses, Anthropic tool-use loop), you may prefer to instrument *that* framework directly for evaluation rather than reimplement its behavior in Inspect. The reproducibility argument is symmetric: the eval should exercise the same agent your users hit, not a re-implementation.

For everything else — internal capability probes, task-family exploration, cross-model comparison on custom tools — Inspect is the right first choice.

## Summary

Inspect gives you the agent-eval scaffolding for free: a `basic_agent` loop that dispatches tool calls and enforces a message budget, a per-sample `store()` for stateful sandbox seeding, first-class Docker sandboxes for tasks that need process isolation, a JSONL dataset loader, and — most importantly — an eval-log format that captures every sample's full trajectory and can be re-scored without re-inferring. The Chapter 2 trajectory vector drops out of a well-designed custom scorer: `Score.value` for final-answer correctness, `Score.metadata` for per-step tool-call stats and termination reasons, and Inspect's built-in token / latency logging for budget rollups. The KV-lookup skeleton in this chapter generalises to SWE-bench, WebArena, and GAIA by swapping tools, sandbox, and success criterion; the shape is the same. Reproducing a public benchmark number should use the benchmark's reference harness; Inspect is the right harness for internal agent evals where you own the tool surface and the task specification.

That closes the module. mod-109 picks up agent evaluation from the safety side (dangerous-capability evals, autonomy, prompt-injection robustness); mod-110 picks it up from the production side (online regression testing of shipped agents). Everything here is the offline-capability instrument that both of those chapters assume you can build.
