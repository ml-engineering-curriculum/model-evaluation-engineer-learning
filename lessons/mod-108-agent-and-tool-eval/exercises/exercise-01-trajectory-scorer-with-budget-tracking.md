# exercise-01: Trajectory Scorer With Budget Tracking

**Estimated effort:** 3 hours

## Objective

Build a reusable trajectory scorer that emits the four-signal vector from Chapter 2 — final-answer correctness, per-step tool-call statistics, budget consumption, termination reason — for any agent run. The scorer is a library: it takes a trajectory (in a small canonical schema) and a task spec, and returns a structured verdict. Its correctness is demonstrated by seeded trajectories with known-good, known-bad, and pathological (looping, budget-exhausting, malformed-tool-call) properties, not by connecting it to a live model.

The point of the exercise is the scorer, not any single benchmark. You will reuse it in exercises 02, 03, and 05.

## Prerequisites

- mod-108 Chapter 2 (trajectory scoring and budget tracking).
- mod-101 Chapter 2 (bootstrap CIs; the aggregator uses trajectory-level bootstrap).
- A Python 3.11+ environment. No sandbox is needed — this exercise uses synthetic trajectories.

## The canonical trajectory schema

Every trajectory the scorer accepts follows this schema (JSONL, one line per message, with a run-level header):

```jsonl
{"kind": "run", "task_id": "toy_001", "model": "provider/model-id@version", "budget": {"max_steps": 20, "max_tokens": 20000, "max_wall_clock_s": 60, "max_dollars": 0.10}, "temperature": 0.2, "seed": 17}
{"kind": "message", "role": "system", "content": "..."}
{"kind": "message", "role": "user", "content": "..."}
{"kind": "message", "role": "assistant", "content": "I'll look it up.", "tool_calls": [{"name": "web_search", "arguments": {"q": "..."}}], "usage": {"prompt_tokens": 200, "completion_tokens": 25}, "latency_s": 0.8}
{"kind": "message", "role": "tool", "name": "web_search", "content": "[...]", "status": "ok"}
{"kind": "message", "role": "assistant", "content": "Answer: 42", "tool_calls": [{"name": "submit", "arguments": {"answer": "42"}}], "usage": {"prompt_tokens": 400, "completion_tokens": 15}, "latency_s": 0.5}
{"kind": "terminal", "reason": "submitted", "final_answer": "42"}
```

Terminal reasons: `submitted`, `budget_exhausted`, `harness_error`, `abort_by_scorer`.
Tool status values: `ok`, `error`, `not_found`, `timeout`.

You may extend this schema if a corner case demands it, but the scorer must handle the base schema as specified.

## Requirements

### Part A — the scorer library

Ship `trajectory_scorer/` as a small library, with at least these modules:

- **`trajectory_scorer/schema.py`** — dataclasses for `RunHeader`, `Message`, `Terminal`, `Trajectory`, `TaskSpec`, `TrajectoryScore` matching Chapter 2's `TrajectoryScore`.
- **`trajectory_scorer/parser.py`** — `parse_jsonl(path) -> Trajectory` and the inverse `write_jsonl(trajectory, path)`. Roundtrip is required.
- **`trajectory_scorer/scorer.py`** — the `score_trajectory(trajectory, task, correctness_check)` function. Signature and return type match Chapter 2.
- **`trajectory_scorer/aggregate.py`** — `aggregate(scores: list[TrajectoryScore]) -> ReportSummary` producing accuracy, `no_answer` rate, `harness_error` rate, per-signal means, and cost distribution (mean, median, p95) for tokens, wall-clock, and dollars.
- **`trajectory_scorer/report.py`** — a CLI: `python -m trajectory_scorer.report <scores.json> --out report.md` that renders a Markdown report and a Pareto-style cost-vs-accuracy scatter (matplotlib is fine).

### Part B — the four signals, implemented

Explicitly compute and expose, per trajectory:

- **`final_answer_score`** and **`verdict`** — from `correctness_check`. Verdict is one of `pass`, `fail`, `partial`, `no_answer`.
- **`tool_call_validity_rate`** — fraction of assistant tool calls whose `arguments` (a) parse as JSON when serialized as a string, or (b) already are a dict. Missing or malformed args count as invalid.
- **`tool_execution_success_rate`** — fraction of tool result messages with `status == "ok"`.
- **`redundant_call_rate`** — fraction of tool calls whose `(name, canonical(arguments))` key appeared earlier in the same trajectory. Define your canonicalization (recommended: `json.dumps(args, sort_keys=True)`) and expose it as a helper function.
- **`steps`** — number of assistant turns.
- **`prompt_tokens`**, **`completion_tokens`** — sums of `usage.prompt_tokens` and `usage.completion_tokens` across assistant messages.
- **`wall_clock_s`** — sum of `latency_s` across assistant messages *plus* the tool-execution wall-clock if the trajectory carries it (`tool.latency_s` if present). If not present, use only the assistant latency and note it in the report.
- **`dollars`** — computed from a small provided pricing table (see `pricing.yaml` starter). Missing model in the table → `dollars = None` and a warning.
- **`termination`** — from the `terminal` event's `reason`, or `harness_error` if the trajectory ends without a terminal event.

### Part C — the correctness-check plugins

Ship three built-in `correctness_check` callables in `trajectory_scorer/checks.py`:

- **`exact_match(target: str, normalize=None)`** — returns `(pass, 1.0)` on exact match after optional normalization, else `(fail, 0.0)`.
- **`regex_match(pattern: str)`** — `re.search`-based, verdict from match presence.
- **`stub_predicate(fn: Callable[[Trajectory], tuple[Verdict, float]])`** — a passthrough so a test can inject any predicate.

Later exercises will inject SWE-bench-style graders and rubric-based judges here.

### Part D — the seeded trajectory suite

Ship `tests/trajectories/` with at least seven synthetic trajectory files. Each is a hand-written JSONL following the schema, and each pairs with an assertion in `tests/test_scorer.py`:

1. **`clean_success.jsonl`** — one tool call, correct answer submitted, well under budget. Expected: `verdict=pass`, `validity_rate=1.0`, `execution_success_rate=1.0`, `redundant_call_rate=0.0`, `termination=submitted`.
2. **`clean_failure.jsonl`** — one tool call, wrong answer submitted. Expected: `verdict=fail`, `final_answer_score=0.0`, per-step signals intact.
3. **`redundant_loop.jsonl`** — the same `web_search` call issued 5 times with identical args, then a correct submit. Expected: `verdict=pass`, `redundant_call_rate >= 0.6`.
4. **`malformed_tool_call.jsonl`** — assistant turn whose `arguments` field is a raw string that fails JSON parsing, followed by a valid call and a submit. Expected: `validity_rate < 1.0` and no crash.
5. **`tool_errors.jsonl`** — three tool calls, two of which return `status="error"`. Expected: `execution_success_rate ≈ 0.33`, `verdict` set by the final answer.
6. **`budget_exhausted.jsonl`** — trajectory ends without a `terminal` submitted; last event is `{"kind": "terminal", "reason": "budget_exhausted"}`. Expected: `verdict=no_answer`, `termination=budget_exhausted`.
7. **`harness_error.jsonl`** — trajectory has no terminal event at all (truncated log). Expected: `verdict=no_answer`, `termination=harness_error`.

A scorer that does not pass every one of these probes is a scorer whose per-step numbers you should not trust.

### Part E — aggregation and reporting

Given a list of `TrajectoryScore`s (from a run over 30–100 seeded trajectories in a `runs/` directory you assemble), `aggregate()` must produce:

- **Accuracy** with a 95% CI (bootstrap over trajectories, at least 2,000 resamples; note the seed).
- **Rate of each termination reason.**
- **Per-signal means** (validity, execution success, redundancy) with bootstrap CIs.
- **Cost distribution:** mean, median, p95 for tokens, wall-clock, and dollars.
- **Pareto scatter data:** for each trajectory, a `(dollars, verdict==pass)` point, plus a per-slice breakdown if `TaskSpec.slice` is set.

The `report.py` CLI writes:

```
# Trajectory eval report

Model: provider/model-id@version
Budget: max_steps=20, max_tokens=20000, max_wall_clock_s=60, max_dollars=0.10
Trajectories: 78

## Headline
- Accuracy: 0.61 (95% CI 0.51-0.71, bootstrap n=2000, seed=17)
- No-answer rate: 0.14
- Harness-error rate: 0.02

## Per-step signals
- Tool-call validity rate: mean 0.94 (CI 0.90-0.97)
- Tool-execution success rate: mean 0.81 (CI 0.75-0.87)
- Redundant-call rate: mean 0.09 (CI 0.05-0.14)

## Budget
- Mean tool calls per trajectory: 6.2 (median 4, p95 18)
- Mean completion tokens per trajectory: 1,340 (median 900, p95 5,200)
- Mean wall-clock per trajectory: 12.4 s (median 8.0, p95 42.1)
- Mean dollars per trajectory: $0.0043 (median 0.0028, p95 0.017)

## Termination reasons
- submitted: 68%
- budget_exhausted: 14%
- harness_error: 2%

## Per-slice breakdown
...
```

## Starter guidance

- **Start with the schema and the tests.** Get all seven probes passing on a trivial `score_trajectory` that returns fixed values, then wire up the real computation one signal at a time. If you start with real trajectories from a live agent run, you will spend the whole exercise debugging the model instead of the scorer.
- **Canonicalization of tool-call args is a design choice.** JSON-sort with a stable separator is the recommended default. If your trajectories carry rich types (e.g. nested tool schemas), document what you canonicalize and what you don't. Two calls that differ only in the order of keys should be considered redundant; two calls that differ in a semantically-meaningful arg (a search query, a file path) should not.
- **`no_answer` vs `fail` is load-bearing.** Do not collapse them. A trajectory whose terminal reason is `budget_exhausted` is `no_answer`; a trajectory that submitted a wrong answer is `fail`. If you conflate them your report loses the diagnostic that separates "model would have solved it with more budget" from "model can't solve it."
- **Bootstrap over trajectories, not over messages.** The unit of independence is the trajectory. If you accidentally bootstrap per-message, your CIs will be far too tight.
- **Cost handling.** Some providers omit token counts on tool-result messages, some return them only in aggregate. The scorer should sum whatever is present and log a warning if fields are missing. `dollars` is optional; produce it when the model is in the pricing table, `None` otherwise.
- **Do not treat this as a spec for a real trajectory format.** Real formats (Inspect eval logs, OpenAI Responses, SWE-agent traces) differ. Later exercises will ship *adapters* into this canonical schema; the point of the exercise is that the scorer is agnostic to the source of the trajectory.

## Acceptance criteria

- The scorer library passes all seven seeded probes in `tests/test_scorer.py`.
- The Chapter 2 vector (final answer, four per-step signals, budget rollup with median/p95, termination reason) is emitted per trajectory.
- The aggregator computes bootstrap CIs over trajectories with a fixed seed; the CIs are reproducible.
- `python -m trajectory_scorer.report runs/*.json --out report.md` produces the report shape shown above plus a Pareto plot (cost vs accuracy).
- The bundle contains a `README.md` with rerun instructions and a `pricing.yaml` covering at least two model providers.

## Stretch goals

- **Tool-choice correctness signal.** Add an optional `expected_tool_sequence` field to `TaskSpec` and a `tool_choice_rate` metric that measures fraction of tool calls that appear in the expected set. Do not make this mandatory — it only works when a reference solution exists.
- **`pass^k` reporting.** Extend the aggregator to accept multi-epoch runs (`k` trajectories per task) and compute `pass^k` (all `k` trajectories per task must pass) alongside `pass@1`.
- **Slicer.** Add a `Slice` abstraction to `TaskSpec.metadata` and produce a per-slice section of the report with per-slice CIs. mod-103's per-slice reporting discipline transfers directly.
- **JSONL streaming.** Make the scorer handle a stream of trajectories (`stdin`, or a chunked file) so an eval that produces trajectories one at a time can be scored incrementally.
- **Adapter for Inspect eval logs.** Write `trajectory_scorer/adapters/inspect_log.py` that ingests an Inspect `.eval` file and yields canonical-schema trajectories. Verify with a small Inspect run (the KV-lookup task from Chapter 6 is a good target) that the scorer produces the same headline numbers as Inspect's own aggregator.
