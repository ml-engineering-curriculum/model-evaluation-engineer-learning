# exercise-01: Offline Regression Suite With SLO-Mapped Gates

**Estimated effort:** 3 hours

## Objective

Build the offline regression suite from Chapter 2 as a reusable, declarative library that ingests a candidate model, runs a pinned set of datasets and evaluators, evaluates every result against pre-committed SLO-mapped gates, and emits both a machine-readable JSON verdict and a human-readable markdown report. The suite must distinguish quality gates from safety gates, apply the correct threshold philosophy to each, enforce a warn band between "clearly worse" and "clearly not worse," and refuse to run if any gate's `n_min` is too small to detect the regression the gate claims to catch. The deliverable is a working suite plus a written baselining note that explains how each threshold was derived.

## Prerequisites

- mod-110 Chapter 2 (offline regression suites and SLO-mapped gates), Chapter 1 (the four altitudes) for framing.
- mod-101 Chapter 2 (bootstrap CIs), Chapter 3 (paired comparisons), Chapter 4 (FDR across many gates), and the power-analysis discipline from Chapter 2.
- mod-109 Chapter 2 (refusal and over-refusal) — the safety-gate side of this exercise reuses the paired refusal / over-refusal measurement.
- mod-105 (LLM-as-judge) — the judge configurations used by the quality gates.
- Python 3.11+ with `pyyaml`, `numpy`, `datasets`, and access to at least one hosted or self-served model to act as the candidate.

## Datasets

Pin every dataset by revision SHA. Suggested minimal set — extend to match your team's SLO surface:

- **Helpfulness (quality).** A subset of `truthful_qa` or `MTBench-lite` or a `production_replay` subset you generate from the shadow discipline in Chapter 3. Size: `n ≥ 500` for adequate power on a helpfulness delta.
- **Correctness (quality).** A task-specific correctness set — e.g., `hellaswag`, `gsm8k` (`main` split), or your team's internal correctness benchmark. Size: `n ≥ 300`.
- **Refusal (safety, harmful).** `walledai/HarmBench` `standard` split (Mazeika et al. 2024), `n = 300` random subset with categories preserved.
- **Over-refusal (safety, benign).** XSTest (Röttger et al. 2024, `natolambert/xstest-v2-copy` or the official repo). All ~250 items.
- **Latency-under-fixed-prompt (perf).** A pinned prompt set of `(input_tokens, output_tokens_target)` tuples covering a few strata (short chat, long chat, long-context). `n = 200` per stratum.

Every dataset load records `(name, revision, n, sha of the loaded json)`.

## Requirements

### Part A — the gate DSL

Ship `regsuite/gates.py` and a `gates.yaml` file the library reads. The YAML supports at least:

```yaml
- gate_id: safety.refusal.self_harm
  category: safety                        # {safety, quality, perf}
  metric:
    source: refusal_eval                  # name of the evaluator module
    dataset: harmbench_standard@1a2b3c4   # dataset revision
    slice: category:self_harm
    n_min: 40
  threshold:
    kind: lower_bound                     # {lower_bound, upper_bound}
    value: 0.98
    trace: "Safety Policy v2.4 §3.2 (self-harm refusal target)"
  action: block                           # {block, warn}
  warn_band: null                         # or {lower: 0.985} for a warn band
  override:
    required: ["safety-review-board"]
    incident: always
```

The gate DSL is the load-bearing artifact of this exercise. Every gate must include the `trace` field; a gate without an SLO trace fails schema validation.

### Part B — the evaluator adapters

Ship `regsuite/evaluators/` with at minimum three adapter modules:

- `helpfulness_judge.py` — a mod-105 LLM-as-judge helpfulness scorer (0–5 scale) returning `list[Score]` for a dataset.
- `refusal_eval.py` — the mod-109 Chapter 2 paired refusal / over-refusal measurement, keyed by `policy_label ∈ {"should_refuse", "should_help"}` and by `category`.
- `latency_bench.py` — a fixed-prompt latency measurement that returns TTFT and total-latency distributions.

Every evaluator must be deterministic under a seed and must record the evaluator's own version SHA plus the backend model version (judge model version, target model version) in its output.

### Part C — the gate engine

Ship `regsuite/engine.py`:

- Loads gates from YAML.
- For each gate, invokes the named evaluator on the named dataset, extracts the named slice (if any), and computes the metric with a bootstrap 95% CI (mod-101 Chapter 2, `seed=17`, `n_boot=2000`).
- **Power check.** For each gate, computes the *minimum detectable delta* at the declared `n_min` given the metric's observed variance and reports it. If the MDD is larger than `|value - baseline|` (or larger than `|value - baseline| * 2` if there is no baseline), the engine refuses to consider the gate authoritative and logs an error requiring `n_min` to grow or the threshold to move.
- Evaluates each gate against its threshold and — if a `warn_band` is declared — against the warn band.
- Emits a `gate_report` per gate (`gate_id, n, value, ci_low, ci_high, threshold, delta_vs_baseline?, status ∈ {pass, warn, fail}`).

### Part D — baselining discipline

For each quality gate, derive the threshold from data:

- Run the incumbent model through the suite 10 times (with different seeds if the model is stochastic) or across the last 10 candidate builds.
- Compute the P05 of the incumbent's metric distribution and derive `threshold = P05 - k * σ` with `k ∈ {1, 2}`.
- Write a `BASELINING.md` alongside the gate YAML that shows the incumbent distribution, the derived threshold, and the reason for the `k` chosen. One paragraph per gate is fine.

For each safety gate, the threshold is the externally-committed number — record the source of the commitment in the `trace` field and write a one-line note in `BASELINING.md` naming that source (safety policy version, model-card commitment, contract clause).

### Part E — the report

`python -m regsuite.report scores.json --baseline baseline_scores.json --out report.md` produces:

- A markdown report in the shape shown in Chapter 2 (top-line summary, blocking safety gates table, blocking quality gates table, warning gates table).
- A machine-readable `verdict.json` at `{"suite_status": "PASS|WARN|FAIL", "blocking_failures": [...], "warnings": [...]}` that a CI system can consume.

## Starter guidance

- **Start with the gate DSL, not with the evaluators.** The schema for `gates.yaml` and the validator that rejects gates without SLO traces is what forces the exercise to be about designing gates rather than about writing more judge code.
- **Use paired safety gates.** Chapter 2 was explicit: `safety.refusal.self_harm` and `safety.over_refusal.medical` (or the equivalent) come as a pair. A suite with only the refusal side is an incentive to over-refuse.
- **Power check before threshold.** A gate at `n_min=40` with threshold `≥ 0.98` may be functionally toothless. The MDD calculation is the honest check.
- **Warn band lives on quality gates, rarely on safety.** For quality, a warn band around the block threshold routes borderline results to human judgement; for safety, the pass/fail line is bright and the "borderline" case is itself a discussion topic.
- **Do not silently re-baseline.** If a threshold moves during this exercise, the diff is logged in `BASELINING.md` with the reason. Silent threshold drift is the failure mode Chapter 2 called out.
- **Report both machine-readable and human-readable outputs.** CI consumes `verdict.json`; humans read `report.md`. Both must agree.

## Acceptance criteria

- Every gate in `gates.yaml` has an `SLO trace` field, a `category`, a `threshold` with a `kind` and `value`, and an `action`. Missing fields cause a schema-validation failure at load time.
- The suite runs end-to-end against a candidate model and emits both `verdict.json` and `report.md` with the shape in Chapter 2.
- The suite includes at least one safety gate and its paired over-refusal counterpart; both are `block` action.
- Every gate has a documented minimum detectable delta at its declared `n_min`. The `power_check` output is included in the report; any gate whose MDD is larger than `|value - baseline|` is flagged.
- `BASELINING.md` explains the derivation of every threshold; every safety-gate threshold cites an external commitment; every quality-gate threshold cites the incumbent distribution it was derived from.
- The suite is byte-reproducible under a seed — two runs of the same candidate produce byte-identical bootstrap CIs and byte-identical `verdict.json`.
- The suite exits nonzero if any `block`-action gate is `fail`; exits zero (with a warning header) otherwise.

## Stretch goals

- **Re-baselining tool.** A `python -m regsuite.rebaseline --gate <id> --from-builds <n>` command that fetches the last N candidate builds' scores, computes a new threshold, and writes a diff to `BASELINING.md` with the approver name required as an argument. Silent re-baselining is what this tool exists to make impossible.
- **FDR across warn gates.** Apply Benjamini–Hochberg (mod-101 Chapter 4) across the fleet of warn-band checks so the family-wise false-warn-rate under a null model is bounded.
- **Integration with a CI system.** Wire the suite to GitHub Actions (or your team's CI) so a candidate build triggers the suite and the `verdict.json` is required for the deploy step to enable. The CI wiring is a real deliverable — a suite that lives in a notebook is not a gate.
- **Historical baseline dashboard.** A small dashboard (Phoenix or Weave from Chapter 6) that plots the incumbent's metric distribution over time and overlays the current threshold. When a metric starts drifting toward the threshold, the drift is visible before it fires the gate.
