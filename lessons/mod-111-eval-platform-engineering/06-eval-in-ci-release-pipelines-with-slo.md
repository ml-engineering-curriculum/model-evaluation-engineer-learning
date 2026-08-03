# Eval in CI, Release Pipelines, and the Platform's Own SLOs

The last layer of the eval platform is the seam where it meets the systems that actually ship models: the CI system that runs on every pull request, the release pipeline that promotes a candidate through offline gate → shadow → A/B → full ramp, and the deploy tooling that turns a "release candidate" into "the model behind the production route." If the eval platform is well-designed and the CI/release integration is thoughtless, the platform's value evaporates — evaluations run but nobody blocks on them, gate reports get emailed but never read, and the model that ships is the one whose author found a way to route around the checks.

This chapter is about that integration and about the platform's own operational commitments. Two threads run through it: how the CI/release systems consume the platform's outputs (the gate integration), and what the platform owes those consumers in return (availability, latency, correctness, cost — the platform-side SLOs).

## Where the platform plugs into CI and release

Three plug-in points recur across organizations. Each targets a different velocity and a different consequence.

### Pull-request-level CI: fast smoke tests, not gates

Every pull request to the model repo (or the eval-config repo, or the judge-registration repo) triggers a lightweight CI job that validates the eval configurations. This is *not* the release gate — it is the smoke test that catches configuration errors before they land on `main`.

Contents:

- **Schema validation for registry additions.** A PR that adds a new judge whose backend model is unversioned fails here. A PR that adds a task whose prompt template references undeclared variables fails here.
- **A short sanity-run against a tiny fixture dataset.** If the PR touches a task, run the task's evaluator against a 5–10-item fixture. If it errors, the PR fails. This catches "the adapter's parse regex was broken" without waiting for the full-scale eval.
- **Diff-aware re-baseline warning.** If the PR touches a judge whose consumers include an active release-blocking gate, the CI job attaches a comment listing the affected gates and requires an explicit reviewer acknowledgment before merge.

These jobs are fast (target: under 5 minutes) and cheap (single-digit dollars of vendor spend). They don't reach into Chapter 4's expensive judge tiers; the fixture-dataset runs use Tier 0 or Tier 1 judges by default. They fail closed on schema issues and open on smoke-run flakiness (a flaky smoke is not a merge-blocker; the fix is a smoke-run rerun button, not a policy exception).

### Release-candidate promotion: the actual gate

When a candidate model is promoted from the training system into a release-candidate state, the release pipeline invokes the offline regression suite from mod-110 Chapter 2 *through the platform*. This is the release-blocking altitude.

The invocation flow:

1. The release pipeline calls the platform's API — typically `platform.submit_suite(suite_id, model_ref, priority=release_blocking, budget=...)` — and receives a `suite_run_id`.
2. The pipeline polls `suite_run_id` for status or subscribes to a webhook.
3. On completion, the pipeline retrieves the `verdict` — a small JSON of `{"status": "PASS|WARN|FAIL", "blocking_failures": [...], "warnings": [...]}` — and gates on it.
4. On fail, the release pipeline halts and surfaces the report URL. On pass, the pipeline proceeds to the shadow altitude (mod-110 Chapter 3).

Three properties of this integration matter:

- **The pipeline gates on `verdict.status`, not on individual metrics.** Only the platform knows the full gate DSL; the pipeline treats `verdict` as opaque and does not try to reason about individual metric values. That property is what lets gate definitions evolve in the registry without every release pipeline having to be re-wired.
- **The pipeline pins the suite revision.** `suite_id` resolves to a Chapter 2 registry entry, and the pipeline records the resolved hash in the release audit trail. A silently-edited suite would land a different set of gates on a release without provenance; the pipeline refuses to accept a suite that has been mutated in place.
- **Override is a named process.** When a gate blocks and the release manager wants to override, the override is a first-class API call (`platform.override_verdict(suite_run_id, override_id, actor, reason)`), not a wave-through. Chapter 2 of mod-110 established the discipline; Chapter 6 wires it into the tooling. Every override lands as a row in the warehouse alongside the run it overrode.

### Post-deploy: continuous eval and drift monitoring

Once the model is behind a production route, the eval program moves to mod-110 Chapters 5 and 6 — sequential monitors on live traffic, drift alerts from the observability platform. The eval platform's role at this altitude is to be the data source and the compute layer for the recurring checks:

- **Scheduled runs.** The `scheduled-refresh` workload class from Chapter 4 executes recurring baseline runs — nightly regression on a subset of the offline suite; weekly re-baseline computation; monthly cost-attribution rollup.
- **Alert-triggered runs.** When the observability platform's sequential monitor fires an alert, the runbook typically escalates by running the offline regression suite against the current production model to see whether the metric shift is a real regression or an input-drift artifact. The eval platform is invoked from the runbook via the same API surface as the release pipeline.
- **Ad-hoc replays.** An investigator can invoke `platform.replay(run_id)` to reproduce a historical run against a different model (candidate for an A/B, upgraded judge for a re-baseline). The replay uses the same registry-resolved artifacts as the original.

The post-deploy integration is where the platform earns its keep at scale — every mod-110 altitude beyond the offline gate becomes cheap when the platform is doing the reproducible-execution and warehouse-write work.

## The eval-in-CI failure modes to design against

Two anti-patterns recur across organizations and are worth naming so that the integration above is understood as the alternative to them.

### Failure mode: eval runs inside the CI job's own process

The release pipeline's YAML has a step that runs `python run_evals.py --model=...`. The evaluator lives in the CI runner's Python environment, the results are written to a temp directory in the CI job's workspace, and the pass/fail decision is made by a `sys.exit(1)` in the script.

Consequences:

- The CI runner is a poor place to run a 2-million-token eval — CI runners have small budgets, short timeouts, and rate-limit awareness that is roughly nil.
- The results are not in the warehouse. The next release has no historical comparison to draw against.
- The CI job's Python environment drifts silently; a `pip install` in the runner is a re-baseline event that no one noticed.
- Cost attribution is impossible because the vendor bill lands under the CI system's shared account, not the eval program's.

The fix is the pattern above: the CI job calls the platform's API. The eval runs on the platform's compute, uses the platform's budgets, writes to the platform's warehouse, and returns a small verdict.

### Failure mode: the release-blocking gate is a green light that never turns red

The pipeline's YAML has an eval step that unconditionally emits `pass` because the actual gate mechanism was never wired up (or was wired up and disabled after a false-positive during an incident). The `Eval: passed` badge appears on every PR and no one has any reason to look twice.

Consequences:

- The platform's gate reports pile up in the warehouse; nobody consumes them.
- The false-positive that motivated the wire-off is never resolved; the discipline is quietly dead.

The fix is process, not code: the platform enforces that release-blocking runs' verdicts are consumed. Chapter 5's warehouse lets the platform audit "which runs' verdicts were `fail` but the release proceeded anyway" — if that count is nonzero and there are no matching override records, the deployment tooling has a bug and the platform's SLO on correctness (below) is violated.

## The four platform SLOs

An eval platform's consumers depend on it for release-critical decisions; that dependency deserves a documented commitment. The four categories introduced in Chapter 1:

### Availability

The percentage of eval-run requests submitted over the SLO window that reach a terminal state (`succeeded`, `failed`, `cancelled`, `budget_exceeded`) without infrastructure error (`platform_error`).

Practical targets:

- **99.0% monthly for `release-blocking` workload class.** Two-nines is what release-critical batch systems usually justify — five-nines would require redundant compute for load that is mostly bursty and unpredictable.
- **99.5% monthly for `interactive`.** A person's confusion tolerance is lower than a batch system's.
- **99.0% weekly for `scheduled-refresh`.** These are the recurring baseline runs; occasional weekly misses are acceptable if they self-heal within a few days.

Error budget: platform errors that consume the budget are things the platform is at fault for — queue crashes, worker OOMs, ingestion failures, orchestration plane 500s. Vendor rate-limits and upstream 429s that are handled by the backoff/retry policy are *not* platform errors as long as the run eventually completes.

### Latency

Wall-clock time from `submitted_at` to `completed_at`, stratified by workload class and by expected run size.

Practical targets:

- **`release-blocking` P95 ≤ 60 min for the standard offline suite.** Derived by measuring the current incumbent's suite runs and setting the SLO at their P95 plus a small margin.
- **`interactive` P95 ≤ 3 min for a fixture-scale eval.** Interactive users tolerate seconds, not minutes.
- **`safety-canary` P95 ≤ 8 hours.** Adversarial suites and full red-team runs are longer; they still have a deadline (before the launch review).
- **`research-sweep` P95 ≤ declared budget wall-clock.** Sweeps declare their own wall-clock budget at submission; the platform's SLO is that it stays within the declared budget, not a fixed number.

The SLO on latency composes with Chapter 4's priority and preemption mechanics. Reserved concurrency for `release-blocking` is *how* the SLO is met, not a promise separate from it.

### Correctness

Two sub-metrics:

- **Reproducibility.** The percentage of completed runs whose `platform.rerun(run_id)` produces byte-identical `run_aggregate_metric` values on the aggregate side. Target: 99.9% within the payload retention window. Failures are lineage bugs (a missing pin, a runner-version drift, a nondeterministic backend) and each one is a discrete post-mortem.
- **Verdict fidelity.** The percentage of `release-blocking` runs whose `verdict.status` was consumed by the release pipeline as-declared — pass proceeds, fail blocks (or is overridden through the documented process). Target: 100%. Anything less is a broken gate.

Correctness SLOs are the ones that failing quietly is a bug the whole platform exists to prevent; they are not a place for graceful degradation.

### Cost predictability

Two sub-metrics:

- **In-budget completion rate.** The percentage of runs that complete within their declared `budget_tokens`. Target: 95%. Runs that exceed are terminated or downgraded per Chapter 4's mechanics; a sustained shortfall means the projections are miscalibrated and Chapter 5's historical distribution needs a re-fit.
- **Platform-wide budget adherence.** The percentage of monthly billing periods in which the platform's aggregate spend falls within its declared budget. Target: 100% within a stated tolerance (e.g., ±10%). Overruns are alert-worthy events with an incident review.

Cost SLOs are the ones that align the platform team's incentives with the finance team's — a platform without cost SLOs will drift over time into a shape that is expensive in ways nobody notices until the annual budget conversation.

## Deriving SLO targets from history

Practical targets are not guessed; they are derived from the platform's own historical distribution in the warehouse. Chapter 5's schema makes this a query:

```
SELECT
  workload_class,
  APPROX_PERCENTILE(EXTRACT(EPOCH FROM completed_at - submitted_at), 0.95) / 60.0 AS p95_min,
  COUNT(*)                                                                        AS n_runs,
  SUM(CASE WHEN status IN ('succeeded', 'failed', 'cancelled', 'budget_exceeded') THEN 1 ELSE 0 END)::float
    / COUNT(*)                                                                    AS terminal_state_rate,
  SUM(CASE WHEN status = 'platform_error' THEN 1 ELSE 0 END)::float
    / COUNT(*)                                                                    AS platform_error_rate
FROM eval_run
WHERE submitted_at >= NOW() - INTERVAL '90 days'
GROUP BY workload_class;
```

The rolling P95 and platform-error rate are the empirical distribution the SLO should be set against. A P95 of 42 minutes over 90 days is a defensible 60-minute SLO with an 18-minute headroom; a P95 of 55 minutes is a defensible 75-minute SLO or a signal to invest in capacity. Guessing the target without this query is how eval platforms end up with SLOs that are simultaneously too tight (nobody meets them and the SLO discipline erodes) and too loose (real degradation doesn't fire).

## Error budgets and incident escalation

Once the SLOs are set, the platform runs an error-budget discipline analogous to any other SRE-managed service (Beyer, Jones, Petoff, Murphy 2016 is the standard reference). Two aspects deserve eval-specific treatment.

- **The eval error budget composes with the release-train's error budget.** A month where the platform blew its `release-blocking` availability SLO because of a specific incident should feed into the launch scorecard — mod-112's systems-design capstone treats "eval was down for four hours during a launch window" as a first-class launch risk. It is not just an SRE concern.
- **Silent failures do not consume the error budget by default and this is dangerous.** A run that returned `succeeded` but produced a metric that later turned out to be wrong (a bad prompt template, a mis-routed judge tier) is a correctness failure that the vanilla monitoring will miss. Chapter 5's aggregate-mismatch lint is what catches these; the platform treats every mismatch as a partial correctness-budget event even when the run itself was labeled success.

Runbooks for common incident classes: vendor rate-limit spike (backoff aggressively, downgrade `research-sweep` class, alert cost owners); orchestration plane 500s (drain queue, engage on-call, halt release-blocking dispatch); warehouse ingestion lag (accumulate events, catch up, verify aggregate recomputation; do *not* drop events).

## Platform observability: instrument the platform itself

The eval platform is subject to the same observability discipline it hosts for the model traffic. Four categories of instrumentation are load-bearing.

- **Metric time-series.** Per-workload-class queue depth, wall-clock latency percentiles, error rates, cost-per-run, budget-consumption rate. Ship to whatever the org uses (Prometheus, Datadog, an internal metrics store) and dashboard by workload class.
- **Structured logs on every plane action.** Every submission, every dispatch, every retry, every override — a structured log entry with `run_id`, actor, and reason. Chapter 5's warehouse is where these land for analytics; a hot log store (Loki, Elasticsearch, an operational logs backend) is where on-call actually looks.
- **Traces on the orchestration path.** OpenTelemetry traces from `submit → resolve → dispatch → adapter execute → warehouse ingest` so that a latency spike can be attributed to a specific span. This is the same OpenInference/OTel discipline mod-110 Chapter 6 introduced for the model side; the platform benefits from it symmetrically.
- **The platform's own aggregate views.** Weekly rollups of "runs by team," "spend by workload class," "SLO adherence by month," "verdicts overridden by release manager." These are the surfaces platform stakeholders (VP-level, finance, safety review) consume — they don't want the raw dashboard, they want the summary.

## Publishing the SLOs to consumers

An SLO document lives with the platform's user-facing docs — a page (or a `/status`-style endpoint) that says what the current SLOs are, what the current adherence is, and what the escalation path is when a run misses its SLO. Two properties:

- **Adherence is published at the same cadence as the SLO.** Monthly SLOs get a monthly report; weekly SLOs get a weekly report. A monthly SLO that has been silently missed for three months without a public report is worse than not publishing anything.
- **Consumers can subscribe to their tenant's own signals.** A team that submits `release-blocking` runs subscribes to alerts when *their* runs' latency percentiles exceed the SLO — not just the platform's aggregate. Per-tenant signals matter more than aggregate signals because the aggregate can look healthy while a specific tenant is systematically underserved (a queue-priority misconfiguration, a runner adapter that specifically breaks on that tenant's workload shape).

## Failure modes the SLO discipline is designed to prevent

### Failure mode: eval is a bottleneck nobody named

Every release is now half a day later than it was a year ago because the eval suite grew from 20 to 200 gates and the platform's throughput didn't. Nobody has a "why is this slow" conversation because there is no target to compare against; the launch review just accepts the slowness as the new normal.

Mitigation: the wall-clock SLO makes the drift visible. When P95 climbs past the SLO for three consecutive months, that is an alert; the response is to either expand capacity or prune the suite. Either is a decidable action.

### Failure mode: the platform's own bugs are invisible

Runs are being labeled `succeeded` but the aggregate metrics are wrong because a runner adapter regressed. The mismatch has been building for six weeks; nobody noticed because the numbers "look plausible."

Mitigation: the aggregate-mismatch lint (Chapter 5) plus the correctness SLO make the pattern surface. The lint fires on runs individually; the SLO fires when the pattern crosses a threshold. Both surface at review time, not at incident time.

### Failure mode: eval spends drift up and no conversation triggers

Vendor bills climb 20% quarter-over-quarter. Nobody notices because there is no threshold, no attribution, and no owner.

Mitigation: the cost SLO and Chapter 5's attribution query pair to make the trend visible per-team and per-workload-class. The finance conversation is on the calendar, not a surprise.

## Guidance for the platform engineer

- **CI calls the platform; it does not run evals.** The CI runner is a wrong place to run a 2-million-token eval. The platform's compute is right; the CI's role is submission and verdict consumption.
- **Verdict is opaque to the pipeline.** The pipeline gates on `PASS/WARN/FAIL`, not on individual metrics. Gate definitions evolve in the registry.
- **Override is a first-class API call, not a bypass.** Every override lands as a warehouse row alongside the run it overrode. Silent overrides are gate death.
- **SLO targets are derived from the warehouse, not guessed.** Chapter 5's queries are the source; a P95 target sets itself once you look at 90 days of history.
- **Error budget composes with the release train's budget.** Eval downtime during a launch window is a launch risk, not just an SRE issue.
- **Instrument the platform itself.** The eval platform runs its own metric time-series, its own structured logs, its own traces, and its own dashboards. It is a service.
- **Publish SLO adherence at the SLO cadence.** A monthly SLO with no monthly report is a fiction.

## Summary

The last layer of the eval platform is its seam with the CI, release, and post-deploy systems, and the operational discipline that layer requires. CI plugs in at three altitudes: fast schema-validation and smoke tests on every PR; the release-candidate offline suite invocation whose verdict gates promotion; and post-deploy scheduled, alert-triggered, and replay-style runs. The pattern is uniform — the CI system submits through the platform's API and consumes an opaque verdict; the pipeline never executes evals itself. The platform, in exchange, commits to four SLO categories: availability (per workload class), latency (per class, derived from warehouse history), correctness (reproducibility plus verdict fidelity), and cost predictability (in-budget completion plus platform-wide adherence). Targets are derived from the warehouse's own distribution, published at the same cadence as the SLO, and instrumented on the platform itself. Error budgets compose with the release train's — an eval outage during a launch is a launch risk, not just an SRE fact. With this layer in place, the platform delivers what Chapter 1 promised: the nth evaluation is cheaper, faster, and more trustworthy than the first. The exercises walk each layer as a concrete design task; the resources file names the primary references for the tooling and standards this module cited.
