# exercise-04: Sequential Testing and Confidence Sequences for Continuous Monitoring

**Estimated effort:** 3 hours

## Objective

Build the continuous-monitoring altitude of production eval from Chapter 5. Ship a small monitor framework that ingests a stream of per-observation values (proportion or bounded-real), maintains a confidence-sequence CI on the running mean, alerts against pre-committed thresholds, applies a family-wise α discipline across a fleet of monitors, and includes a **judge-canary monitor** as a first-class member of that fleet. Demonstrate correctness with a Monte-Carlo experiment that measures the false-alarm rate empirically under a null and the time-to-detection under a real drift.

The exercise is not about running a real production monitor (it is a codebase, not a service); it is about the statistical correctness of the CI construction, the α budgeting across the fleet, and the runbook plumbing that turns an alert into a diagnostic.

## Prerequisites

- mod-110 Chapter 5 (sequential testing and confidence sequences), Chapter 6 (drift alerts, judge drift), Chapter 4 (pre-registered rules).
- mod-101 Chapter 4 (Benjamini–Hochberg FDR) — the family-wise discipline used in Part D.
- Howard, Ramdas, McAuliffe & Sekhon 2021 (empirical Bernstein CS), Waudby-Smith & Ramdas 2020 (betting-based CS), Johari, Pekelis & Walsh 2017 (always-valid p-values) — reference for the CI construction. Do not re-derive; use the `confseq` Python package (Howard–Ramdas group) or a validated internal implementation.
- Python 3.11+ with `numpy`, `scipy`, and the `confseq` package (`pip install confseq`).

## Datasets

You need two synthetic streams and one grounding dataset:

- **Synthetic Bernoulli stream.** A generator `stream_bernoulli(p, n, drift_at=None, p_after=None, seed=17)` that yields `n` `{0, 1}` observations from `Bernoulli(p)` with an optional level shift at `drift_at`. Used to measure false-alarm rate (stationary) and time-to-detection (drift).
- **Synthetic bounded-real stream.** A generator `stream_bounded(mean, var, n, drift_at=None, mean_after=None, a=0.0, b=1.0, seed=17)` for a judge-score-shaped monitor.
- **Judge-canary dataset.** A pinned set of 50–200 `(prompt, response, gold_label)` triples that a real LLM judge grades. For this exercise, build the canary from a public dataset (e.g., 100 items from `truthful_qa` with human-authored gold "correct / incorrect" labels, or from HELM's `mmlu` split with the multiple-choice answer as the gold). Two annotators is aspirational for the exercise; for the coded deliverable, one careful reviewer of the labels is enough.

Every stream and every dataset load records its seed, size, and — for the judge canary — the judge backend model version and the judge prompt SHA.

## Requirements

### Part A — the CS engine

Ship `monitor/cs.py` wrapping the `confseq` package (or an internal implementation validated against it):

- `class ConfidenceSequence(alpha, kind={"empirical_bernstein", "betting"}, bound=(a, b))` — stateful; `update(value)` appends one observation and returns `(mean_t, lower_t, upper_t, n_t)`.
- Support both `Bernoulli` (for proportions) and `bounded-real` (for judge scores in `[0, K]`).
- Include a unit test that empirically measures the false-alarm rate on `10,000` runs of a stationary stream at `alpha=0.05` and asserts it is bounded by `alpha` (with a small Monte-Carlo tolerance).

### Part B — the monitor DSL

Ship `monitors.yaml` describing each monitor declaratively:

```yaml
- monitor_id: safety.refusal.production.self_harm
  metric_family: safety
  alpha: 0.001
  bound: [0.0, 1.0]
  kind: bernoulli
  threshold:
    kind: absolute_lower_bound
    baseline: 0.996
    alert_if_upper_below: 0.990
  runbook: runbooks/safety-refusal.md
  severity: p1

- monitor_id: quality.helpfulness.production
  metric_family: quality
  alpha: 0.01
  bound: [0.0, 5.0]
  kind: bounded_real
  threshold:
    kind: baseline_relative_lower_bound
    baseline_source: rolling_28d_mean
    delta: -0.05
  runbook: runbooks/quality-drop.md
  severity: p3

- monitor_id: drift.input.query_length_psi
  metric_family: drift
  alpha: 0.05
  bound: [0.0, 5.0]         # PSI-like statistic domain
  kind: bounded_real
  threshold:
    kind: absolute_upper_bound
    alert_if_lower_above: 0.10
  runbook: runbooks/input-drift.md
  severity: p4
```

The DSL is the load-bearing artifact — every monitor names its severity, its runbook, and its α tier.

### Part C — the runner and the alerter

Ship `monitor/runner.py`:

- Loads `monitors.yaml`.
- Ingests a stream per monitor (per-metric per-slice).
- Advances the CS on each observation (or on a downsampled cadence, e.g., every `k` observations, configurable — the false-alarm guarantee still holds; the alert latency is bounded by the cadence).
- Fires an alert when the CI crosses the threshold, calling `alert(monitor_id, severity, runbook_path, current_state)`.
- Suppresses duplicate alerts within a configurable cool-down window (default 1 hour) unless the CI has recovered and re-crossed.
- Writes a per-alert `AlertRecord` with timestamp, monitor id, current mean, CI, threshold, and the runbook path.

### Part D — the FDR discipline across the fleet

For monitors in the same `metric_family`, apply Benjamini–Hochberg to the family's `always-valid p-values` at each observation step (Chapter 5's α-budget-by-family). Implement `family_fdr_wrap(monitors, fdr_alpha)` — the alerter fires per monitor only if the monitor's individual α (from Part B) triggers *and* the family-level BH-adjusted rate does not reject the alert as noise. Ship a unit test that runs 100 monitors in one family under a null and demonstrates the family false-alarm rate is bounded by `fdr_alpha`.

### Part E — the judge-canary monitor

Ship `monitor/judge_canary.py`:

- Loads the judge-canary dataset (Part above).
- On a schedule (in the exercise, a manual `run_once()` call), grades every item with the current judge and computes the **agreement rate** with the gold labels.
- Feeds the per-item `{0, 1}` agreement observations into a `ConfidenceSequence` (α = 0.001, tighter than quality monitors because judge drift is upstream of every quality metric).
- Alerts when the CS upper bound drops below the pre-committed floor (default 0.85 for a judge historically at 0.90+).
- Include a mock-drift test that swaps the judge to a deliberately-worse model mid-run and shows the canary alerts within `k` runs.

### Part F — the runbooks

Every monitor in `monitors.yaml` names a runbook markdown file under `runbooks/`. Each runbook has the shape:

- Symptom (what the alert says).
- Diagnostic steps (a numbered checklist of things to inspect — usually the three drift categories from Chapter 6, judge-canary status, recent releases, upstream traffic anomalies).
- Expected outcomes (each diagnostic step's `X | Y | Z` result and what it points to).
- Actions per outcome (rollback, re-baseline, page-owner, log-and-monitor).
- Owner (the team that gets paged).
- Escalation path.

An alert without a runbook cannot exist in `monitors.yaml` — schema validation fails at load time.

### Part G — the Monte-Carlo validation

Ship `monitor/validate.py` that runs:

- **False-alarm rate under a null.** For each `α ∈ {0.001, 0.01, 0.05}`, generate `10,000` stationary streams (length 10,000), instantiate a fresh CS per stream, and count the fraction of streams in which the CI ever excludes the true parameter. Report empirical false-alarm rate vs. `α`; assert bounded by `α` with a small margin.
- **Time-to-detection under drift.** For a real drift (`p` shifts from `0.99` to `0.97` at step `1,000`), report the median and P90 steps-to-first-alert per `α` and per CS construction (empirical Bernstein vs. betting). Compare to the fixed-`n` alternative to demonstrate the peek-any-time value.
- **α-widening penalty.** Compare the CS half-width at `n = 10,000` to the fixed-`n` Wald / Wilson half-width at the same `n`. Confirm the 20–50% widening penalty from Chapter 5.

## Starter guidance

- **Do not re-derive the CS bound.** The empirical-Bernstein constants are subtle and easy to get wrong. Use `confseq`'s `EBWConfidenceInterval` / `predmix_upper_cs` primitives; if you must implement, cross-check numerically against `confseq`'s output on the same stream.
- **Welford's algorithm for the incremental variance.** The `(sum_sq - n * mean^2) / (n - 1)` formulation from Chapter 5's sketch is numerically fragile; Welford is the correct online-variance recurrence.
- **α per monitor, plus FDR per family.** The two disciplines compose: individual monitors have their own α, and the family-level BH keeps the fleet's aggregate false-alarm bounded.
- **Every monitor has a runbook, always.** The schema check enforces this. An alert without a runbook is noise that pages the on-call for no reason; Chapter 6's alert-fatigue trap is the operational failure this exercise designs against.
- **The judge-canary is the most operationally-valuable monitor in the fleet.** If you build only one monitor from this exercise, build that one.
- **Downsample the CS update cadence if the stream is high-volume.** Updating the CS on every observation is fine mathematically but expensive at production QPS; every 100 or 1000 observations is the practical setting.
- **The alert is a diagnostic trigger, not a rollback trigger.** The runbook drives the action; the CS just says "something moved."

## Acceptance criteria

- CS unit test: empirical false-alarm rate is bounded by `α` (with Monte-Carlo tolerance) at `α ∈ {0.001, 0.01, 0.05}`.
- Monitor DSL loads and validates `monitors.yaml`; a monitor without a `runbook` or a `severity` fails schema validation.
- The runner processes a stream end-to-end and produces `AlertRecord`s that a launch review could actually read.
- Family-level BH across monitors of the same family passes its unit test: 100 monitors under a null produce a family-wise false-alarm rate bounded by `fdr_alpha`.
- The judge-canary monitor detects a deliberate mid-run judge swap within `k ≤ 3` runs on the pinned canary set at `α = 0.001`.
- Every monitor in `monitors.yaml` has a corresponding runbook file with all six sections (symptom, diagnostic, expected outcomes, actions, owner, escalation).
- The Monte-Carlo validation report includes empirical false-alarm rate, median / P90 time-to-detection, and the α-widening penalty at `n = 10,000`.
- The monitor pipeline is reproducible under a seed — two runs of the same simulator config produce byte-identical alert timestamps and monitor states.

## Stretch goals

- **Group-sequential design as a comparison baseline.** Add an O'Brien–Fleming group-sequential test for a fixed-`n_max` monitor and show that under the same drift, its time-to-detection is competitive with the CS but its α is invalid the moment you exceed `n_max`. Discuss when each is the right tool (Chapter 5's "fit the setting").
- **Two-sample CS for a shadow-mode monitor.** Extend the CS to compare two live streams (incumbent vs. candidate) with a paired-difference CS on the delta. This is the shadow-side monitor for a slow-ramping deployment.
- **α re-tuning report.** After running the fleet for a simulated 90 days, produce an audit that lists which monitors fired, how often, and what fraction of alerts were true positives (using the ground-truth drift injections). Recommend α adjustments per Chapter 6's alert-fatigue discipline.
- **Wire into an observability platform.** Emit `AlertRecord`s as annotated spans in Phoenix / Langfuse / Weave (Chapter 6). Ship a dashboard configuration that renders each monitor's CS trace with the threshold line and the last alert timestamp.
- **Judge-canary calibration.** Extend the judge canary with two-annotator gold labels, compute Cohen's κ (mod-106 material), and derive the floor for the agreement threshold empirically from the annotator agreement rather than eyeballing 0.85.
