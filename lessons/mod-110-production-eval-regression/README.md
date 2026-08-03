# mod-110-production-eval-regression: Production Evaluation and Regression Detection

**Estimated effort:** 16 hours

## Learning objectives

- Design an offline regression suite with pass/fail gates that map to product SLOs and safety policy
- Run shadow / dark-launch comparisons and read the result correctly under non-IID production traffic
- Design a controlled A/B experiment for a model release with CUPED variance reduction and pre-registered hypotheses
- Apply sequential testing / confidence sequences for safe continuous monitoring and early stopping
- Wire eval into production observability (Arize Phoenix, Langfuse, W&B Weave) and design drift / judge-drift alerts
- Reason about MLPerf-style inference benchmarking for the serving altitude (TTFT, TPOT, throughput vs. accuracy floor)

## Chapters

1. [Why Production Evaluation Is Different](01-why-production-eval-is-different.md) — frames the four altitudes (offline regression, shadow, A/B, continuous monitoring) plus the two supporting systems (observability, serving benchmarks); draws the line between capability measurement and decision-support measurement.
2. [Offline Regression Suites and SLO-Mapped Gates](02-offline-regression-suites-and-slo-gates.md) — the pre-launch gate: SLO trace, threshold empirical derivation, quality-vs-safety gate categories, re-baselining as a named event.
3. [Shadow and Dark-Launch Comparisons on Non-IID Traffic](03-shadow-and-dark-launch-comparisons.md) — side-effect-safe plumbing, cluster-robust CIs, stratification, heavy-tailed serving metrics, green/yellow/red decision rule.
4. [Controlled A/B Experiments with CUPED and Pre-Registered Hypotheses](04-ab-experiments-with-cuped.md) — ten-question pre-registration, randomization unit and SUTVA, sample-size / MDE, CUPED variance reduction, SRM check, novelty and primacy.
5. [Sequential Testing and Confidence Sequences for Continuous Monitoring](05-sequential-testing-and-confidence-sequences.md) — group-sequential tests with alpha spending, confidence sequences (empirical Bernstein, betting), α-budget-by-severity, judge-canary monitor.
6. [Production Observability and Drift Alerts](06-production-observability-and-drift-alerts.md) — Arize Phoenix, Langfuse, W&B Weave; the instrumentation contract; input vs. output vs. metric drift; judge-drift alerting; alert fatigue and runbooks.
7. [MLPerf-Style Serving Benchmarks: TTFT, TPOT, Throughput Against an Accuracy Floor](07-mlperf-style-serving-benchmarks.md) — the four numbers, the four scenarios, accuracy floor discipline, reproducibility manifest, reading vendor numbers.

## Structure

- `01-…md` … `07-…md`: lecture chapters (above).
- `exercises/`: per-exercise prompts. Solutions live in the paired `-solutions` repo.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
