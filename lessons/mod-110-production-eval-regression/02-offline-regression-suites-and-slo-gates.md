# Offline Regression Suites and SLO-Mapped Gates

The first altitude of production eval is the one that catches the release before it ever sees a real user. It is an offline suite: fixed datasets, fixed judges, fixed sampling seeds, run deterministically on every candidate model in CI, with an explicit pass/fail outcome. Its job is not to discover new things about the model — that is what mod-101 through mod-109 evaluations do. Its job is to answer one very narrow question: *does this candidate clear the pre-committed bar on the metrics we owe leadership, customers, and safety partners?* When the answer is no, the release is blocked; when the answer is yes, the candidate advances to shadow (Chapter 3).

The failure modes here are almost never about the metrics themselves — those have been designed and validated in the upstream modules. The failure modes are about the *gate*: gates that measure the wrong thing, gates that fire on noise, gates that never fire because their bar was set below the model's floor, gates that were never mapped to a business or safety commitment and so cannot justify blocking a release when they trip. This chapter is about designing gates that a launch review can defend.

## What an SLO-mapped gate actually is

A gate is a triple: `(metric, threshold, action)`. The metric is a number the offline suite computes. The threshold is a bound (usually a lower bound for quality metrics, an upper bound for latency and cost). The action is what happens when the metric crosses the bound — typically "block the release" or "warn, require review, do not block."

For the gate to be defensible, all three parts must trace to something outside the eval system:

- **The metric traces to a construct that appears in a product SLO or a safety policy commitment.** Not "helpfulness score from our internal judge," but "helpfulness score from our internal judge, which we have validated in mod-101 Chapter 1 as a construct-valid proxy for the user-facing thumbs-up rate that appears in the product SLA at 4.0 out of 5 or higher." The chain must be explicit; a metric with no traceable link to a real commitment cannot justify blocking a release when someone with launch authority pushes back.
- **The threshold traces to the historical performance of the incumbent model or to an externally-committed number.** Not "arbitrary round number that sounds strict," but "the P05 of the incumbent model's rolling 30-day performance on the same eval, minus 2 percentage points to accommodate measurement noise." This is what makes the gate empirically calibrated — it fires on real regressions, not on noise, and it does not fire on a model that merely differs from the incumbent in some way that is within historical variance.
- **The action traces to a release-process step.** Not "the pipeline turns red," but "the pipeline emits a `regression-blocking` status that CI is configured to require green before the deploy button in the release console is enabled, and overriding requires the launch owner plus one member of the safety review team." Without the process wiring, the "gate" is just a red pixel that everyone learns to ignore.

The chain (`SLO or safety commitment ↔ metric ↔ threshold ↔ release-process action`) is the definition of an SLO-mapped gate. Every other kind of "pass/fail check in CI" is a smoke test — useful, but not a gate. Smoke tests catch broken serialization and mis-configured judges; gates catch real regressions.

## The two gate categories: quality gates and safety gates

Gates split cleanly along a severity axis, and the two categories deserve different design discipline.

### Quality gates

Quality gates cover the metrics whose regression is a *product* problem: helpfulness, correctness on your task suite, answer-completeness on your domain benchmark, tone-and-format compliance, latency percentiles at fixed request shape, cost-per-request. These are the metrics you tune for. A regression on a quality gate is bad but usually not catastrophic — it means the release does not ship, the model iterates, and the release ships next week instead.

Design discipline for quality gates:

- **Threshold set as `historical_baseline_p05 - 2 * historical_noise_std`.** The `-2σ` cushion is the standard eyeballed choice; if you have enough historical data to run a proper power calculation, use it (mod-101 Chapter 2's bootstrap variance estimate feeds directly into this). The point is: the threshold reflects the noise of the measurement, not a round number.
- **A warning band above the blocking threshold.** Between "the model is clearly worse" (block) and "the model is clearly not worse" (pass) there is a middle zone where you cannot yet tell. A `warn` action in this zone routes to a designated reviewer (usually the eval owner) who decides whether to gather more data (larger sample, held-out subset, shadow deployment) or to accept the borderline result with an explicit sign-off.
- **A budget for the gate as a whole.** The offline regression suite typically has 20–100 gates. If any one of them can block the release, the noise alone will block routine releases regularly. Chapter 5's discipline (FDR-controlled reporting) applies at the *gate design* stage — set the individual thresholds so that under a null model (candidate == incumbent) the probability of any gate firing in a routine release is below your team's tolerance for false blockers (typically 5–10%).

### Safety gates

Safety gates cover the metrics from mod-109: refusal rate on each policy-violating category, over-refusal rate on the benign-lookalike set, jailbreak ASR on the pinned attacker suite, and any dangerous-capability score that a public risk framework or an internal safety policy commits the organization to. These are the metrics whose regression is a *headline* problem, not a product problem. A refusal-rate regression on the self-harm category that ships is a much larger incident than a helpfulness regression that ships.

Design discipline for safety gates is different, in three specific ways:

- **Thresholds are absolute, not baseline-relative.** For a quality metric, "the model regressed by 2 points" is the failure state. For a safety metric, "the model refuses only 92% of self-harm requests" is the failure state, whether the incumbent was at 91% (worse) or 99% (better). The gate is set to the *externally-committed number* — the safety policy statement, the model card commitment, the enterprise contract's safety clause — not to a `baseline - delta` formula. If the incumbent falls below the committed number too, that is itself a bug that should be fixed on the incumbent side, not a reason to let the candidate slide.
- **Bidirectional gates on the refusal / over-refusal pair.** Chapter 2 of mod-109 was explicit: refusal rate can be over-corrected into over-refusal, and both are problems. A safety gate on refusal alone will incentivize the training team to over-refuse, which then fails as a helpfulness regression that no one catches because the quality gates are graded by capability judges that reward completions. The safety gate must be a *paired* gate: refusal ≥ X *and* over-refusal ≤ Y, both bounds committed.
- **Override authority is different.** A quality gate override is usually the launch owner's call. A safety gate override is not — the mod-109 discipline demands that any exception route through the safety review body (in an org that has one) or, at minimum, requires an incident review whether the model ships or not. The override path is a governance decision, not a launch decision.

The two categories share the same gate mechanism but have different SLO traces, different threshold philosophies, and different override paths. A regression suite that treats them uniformly will either be too lenient on safety or too strict on quality; either failure erodes trust in the system.

## Anatomy of the offline regression suite

An offline regression suite that a launch review can defend has five components:

- **Datasets.** Pinned by revision SHA, versioned in the same repo as the gate definitions. Any change to the datasets triggers a re-baseline of the affected gates — otherwise a dataset drift silently shifts the metric and either the gate fires spuriously or it stops firing when it should. Every gate declares its dataset revision explicitly.
- **Metric evaluators.** The same mod-105 judge configurations that produced the metric in the first place, pinned by prompt SHA and model version. A judge upgrade (e.g., moving from `claude-3-5-sonnet-20240620` to `claude-3-5-sonnet-20241022`) is a re-baseline event — see the judge-drift discussion in Chapter 6.
- **Sampling and seeds.** Fixed, reported, and byte-reproducible. If the same candidate model is run through the suite twice on the same day and the metrics differ by more than measurement noise, the suite is broken as a gate — its outputs cannot be trusted for blocking decisions. mod-101 Chapter 2's variance-quantification discipline applies here; the bootstrap CI on any metric must be tight enough that the gate threshold is outside the CI in normal operation.
- **Gate definitions.** Declarative — a YAML or JSON file (or code that lives in the same repo) that names every gate, its metric, its threshold, its action, and — critically — its SLO trace. A gate without a documented SLO trace is a bug in the suite and should be either connected to a commitment or removed.
- **Report and CI integration.** A machine-readable report (JSON) plus a human-readable report (markdown), attached to the release-candidate's build in CI, plus an aggregate `pass / warn / fail` status that the release-management tooling consumes.

A minimal gate definition:

```yaml
- gate_id: safety.refusal.self_harm
  category: safety
  metric:
    source: mod109_refusal_eval
    dataset: harmbench_standard@rev:1a2b3c4
    slice: category:self_harm
    n_min: 40
  threshold:
    kind: lower_bound
    value: 0.98
    trace: "Safety Policy v2.4 §3.2 (self-harm refusal target)"
  action: block
  override:
    required: ["safety-review-board"]
    incident: always

- gate_id: quality.helpfulness.mean
  category: quality
  metric:
    source: internal_judge_v1_2
    dataset: production_replay@rev:d4e5f6
    slice: overall
    n_min: 500
  threshold:
    kind: lower_bound
    value: 4.05
    trace: "Product SLA §2.1 (helpfulness-4.0 target) with -0.05 measurement cushion"
  action: block
  override:
    required: ["launch-owner"]
    incident: none

- gate_id: quality.latency.p95_completion_tokens_512
  category: quality
  metric:
    source: serving_perf_bench
    dataset: fixed_prompt_set@rev:9a8b7c
    slice: completion_tokens=512
    n_min: 200
  threshold:
    kind: upper_bound
    value_ms: 1800
    trace: "Product SLA §4.3 (P95 chat-response TTFT+completion budget)"
  action: warn
```

The three gates above illustrate the pattern: safety gate is `block` with a safety-board override; quality helpfulness gate is `block` with launch-owner override; latency gate is `warn` because a two-second P95 is a product concern but not a shipping blocker until it becomes chronic.

## Baselining: where the numbers come from

The single most common failure mode in a regression suite is thresholds that were guessed. Two failure modes chain from there: thresholds too tight (routine releases fail the gates and everyone learns to override them) or thresholds too loose (real regressions clear the gates and ship silently).

The discipline is to derive every threshold from data:

- **For quality metrics against an incumbent baseline**, compute the incumbent's metric on the fixed dataset over the last N candidate builds (N ≥ 10, ideally 30). The threshold is the P05 (or a matching lower percentile) minus a `k * σ` cushion, where `σ` is the mod-101 bootstrap standard error and `k ∈ {1, 2}` depending on your tolerance for false blockers. Publish the historical distribution alongside the gate definition; when someone asks "why is the threshold 4.05 and not 4.10?", the answer is a chart.
- **For safety metrics against an externally-committed number**, the threshold *is* the commitment. If your policy statement says the model refuses on self-harm at ≥ 98%, the gate is `≥ 0.98`. If the incumbent's floor is above 0.98, you have headroom; if it is below, you have a pre-existing safety debt that is separately actionable.
- **For latency and cost gates**, the threshold traces to the product SLA. If the SLA promises P95 < 2 seconds, the gate is `≤ 1.8s` with the 200ms measurement cushion documented. The cushion covers measurement variance (Chapter 7's serving-bench methodology) and the difference between offline-fixed-prompt latency and real-traffic latency (which is what the SLA cares about).

Re-baselining is a specific event, not an ongoing drift. When the incumbent model changes, when the dataset changes, when the judge upgrades, when the hardware changes — the gate is re-baselined *once*, the new threshold is documented with the reason, and the change is reviewed by the same authority that approves gate overrides. Silent re-baselining ("the gate kept failing so I raised the threshold") is the same bug as silent override; the suite loses its blocking authority.

## The three ways a gate quietly stops working

Three failure modes come up repeatedly.

### Failure mode: the metric drifts while the threshold stays fixed

An LLM-as-judge is upgraded, the dataset gains new items in a routine refresh, the sampling temperature moves from 0.0 to 0.2 for consistency — and the metric shifts by a small amount that is enough to move the incumbent's distribution across the threshold. Gates start firing on the incumbent, someone overrides, and the discipline breaks.

Mitigation: **any change that could shift the metric is a re-baseline event, and the CI system rejects a candidate whose metric-config SHA does not match the SHA the current thresholds were baselined against.** A re-baseline requires re-computing the incumbent's distribution and re-deriving the thresholds. The re-baseline audit trail (old threshold, new threshold, reason, approver) lives with the gate definition in version control.

### Failure mode: the sample size is too small to detect the regression the gate claims to catch

A gate says `n_min: 40` and threshold `≥ 0.98`, meaning "block if refusal rate on 40 self-harm items falls below 98%." Forty items with a Wilson 95% CI on a 0.98 proportion has a half-width of about 3 percentage points — a candidate at 0.95 will still pass the gate over half the time by chance. The gate looks strict but is functionally toothless.

Mitigation: **compute the effective detectable delta for every gate at the declared `n_min`, and either grow `n_min` until the delta is smaller than the regression you actually care about catching, or lower the threshold to the largest value that has statistical power at the current `n_min`.** This is the mod-101 power-analysis discipline applied to the gate design step. Every gate in the suite should have a documented "smallest regression this gate can catch" number.

### Failure mode: the SLO trace is aspirational, not committed

The gate definition says `trace: "Product SLA §2.1 (helpfulness-4.0 target)"`, but the actual SLA does not contain that clause — it was proposed in a doc six months ago, never finalized, and everyone who was in the room has since moved teams. When a launch review pushes back on the gate, no one can point to the commitment.

Mitigation: **every SLO trace must resolve to a document that a launch review can pull up and read: a signed customer contract, a public model card commitment, a safety policy that the safety review body has approved.** During gate design, the SLO trace is validated with the owning function (product for SLAs, safety review for policy). Traces that cannot be validated are either fixed (get the commitment written down and approved) or the gate is downgraded from `block` to `warn`.

## Reporting: the shape of the offline regression report

A candidate model's offline regression report is a machine-readable artifact plus a rendered summary. The summary shape:

```
# Offline Regression Report — release-candidate abc123

Candidate: internal/candidate-v0.14.2
Incumbent: internal/prod-v0.14.1
Suite:     regression-suite@rev:11ff22aa (2026-07-18)
Judge:     internal-judge-v1.2 (prompt SHA 33bb44cc, backend claude-3-7-sonnet-20250219)

## Summary
- Gates evaluated: 43
- Blocking failures: 0
- Warnings: 2
- Passes: 41
- Overall: PASS

## Blocking gates (safety)
| gate_id                              |     n | value | threshold | Δ vs incumbent |    status |
|--------------------------------------|-------|-------|-----------|----------------|-----------|
| safety.refusal.weapons               |    60 |  1.00 |     ≥0.99 |          +0.00 |      PASS |
| safety.refusal.self_harm             |    40 |  0.98 |     ≥0.98 |          -0.02 |      PASS |
| safety.over_refusal.medical          |    60 |  0.10 |     ≤0.15 |          -0.02 |      PASS |
| ...                                  |       |       |           |                |           |

## Blocking gates (quality)
| gate_id                              |     n | value | threshold | Δ vs incumbent |    status |
|--------------------------------------|-------|-------|-----------|----------------|-----------|
| quality.helpfulness.mean             |   800 |  4.11 |     ≥4.05 |          +0.03 |      PASS |
| quality.correctness.tool_use         |   300 |  0.87 |     ≥0.84 |          +0.01 |      PASS |
| quality.correctness.long_context     |   150 |  0.71 |     ≥0.72 |          -0.02 |      WARN |
| ...                                  |       |       |           |                |           |

## Warning gates
| gate_id                              |    detail                                        |
|--------------------------------------|--------------------------------------------------|
| quality.correctness.long_context     | 0.71 within warn-band [0.70, 0.72), incumbent 0.73 |
| quality.latency.p95                  | 1.72s (threshold 1.80s), +80ms vs incumbent      |
```

Two properties of this report matter for the launch review:

- **Every gate line names its threshold and the delta versus incumbent.** A reviewer can immediately see whether "PASS" is comfortable or grazing the threshold. A grazing pass on a safety gate is itself a discussion topic, even though it does not block.
- **Warnings are visible on the top-line report, not buried in an appendix.** The warn-band is where humans exercise judgement; it must show up where humans look. A warning that is only in a log file is a warning that has been silently absorbed.

## The regression suite as a living system

The suite is not a static artifact — it evolves as the product evolves. Three ongoing operations keep it useful:

- **Add a gate when a new SLO or safety commitment is made.** Every time product commits to a customer-facing quality claim, or safety commits to a new policy category, the corresponding gate lands in the suite before the next release. This is the "prevention" pattern from post-mortem discipline: an incident becomes a gate becomes a permanent regression check.
- **Prune a gate when the SLO it traces to is retired.** Dead gates accumulate and dilute the signal. A gate whose SLO is no longer in force is either connected to a new SLO or removed. The audit trail records why.
- **Re-baseline on schedule.** Even without a discrete "the judge changed" event, gate thresholds drift as the product's incumbent floor moves. A quarterly re-baseline (compute the incumbent's distribution over the last quarter's builds, re-derive thresholds, review) prevents thresholds from silently becoming easier or harder than intended.

## Guidance for the eval author

- **No gate without an SLO trace.** If you cannot name the commitment that would be broken by the regression the gate is catching, remove the gate or downgrade it to `warn` until the commitment is in place.
- **Threshold every gate against measurement power.** Every gate has a documented "smallest regression this catches" — if that number is bigger than what you actually care about catching, `n_min` is too small.
- **Bidirectional safety gates.** Refusal without over-refusal is a training-team gaming target. Pair them, always.
- **Warn-band before block.** The uncertainty zone between "clearly worse" and "clearly not worse" is where judgement lives; give it a name and a workflow instead of forcing every candidate into pass or fail.
- **Re-baseline is a named event.** Silent threshold changes destroy the suite's authority. Every re-baseline has an approver, a reason, and a diff.
- **Machine-readable output feeds release automation.** Humans read the markdown; the release console reads the JSON. Both must agree.

## Summary

The offline regression suite is the first altitude of production eval and the only one that can block a release before real traffic sees it. Its unit is the SLO-mapped gate: a metric traceable to a product SLO or safety commitment, a threshold empirically derived from either historical incumbent performance or an externally-committed number, and a release-process action wired into CI. Gates split into quality gates (baseline-relative, launch-owner override) and safety gates (commitment-absolute, safety-board override, always paired for refusal / over-refusal). The three failure modes to design against are metric drift with fixed thresholds, sample sizes too small for the delta the gate claims to catch, and SLO traces that name commitments that were never made. A defensible suite is a versioned artifact with declarative gate definitions, judge/prompt/dataset SHAs, machine-readable output, and re-baselining that is a named event with an approver. The next chapter takes a passed candidate to the second altitude: shadow / dark-launch comparison against real, non-IID production traffic.
