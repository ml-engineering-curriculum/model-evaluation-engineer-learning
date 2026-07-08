# mod-110-production-eval-regression: Production Evaluation and Regression Detection

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Design an offline regression suite with pass/fail gates that map to product SLOs and safety policy
- Run shadow / dark-launch comparisons and read the result correctly under non-IID production traffic
- Design a controlled A/B experiment for a model release with CUPED variance reduction and pre-registered hypotheses
- Apply sequential testing / confidence sequences for safe continuous monitoring and early stopping
- Wire eval into production observability (Arize Phoenix, Langfuse, W&B Weave) and design drift / judge-drift alerts
- Reason about MLPerf-style inference benchmarking for the serving altitude (TTFT, TPOT, throughput vs. accuracy floor)

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
