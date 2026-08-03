# Shadow and Dark-Launch Comparisons on Non-IID Traffic

The offline regression suite from Chapter 2 answers "did the candidate regress on the metrics we committed to?" using pinned datasets. It does not answer "how will the candidate behave on the actual distribution of requests our users send us today?" The two questions are different because the offline dataset is fixed and IID by construction; production traffic is neither. Shadow and dark-launch comparisons close that gap: the candidate model receives real production requests without its output being shown to a user, so the comparison happens on the true request distribution while the user experience remains served entirely by the incumbent.

This altitude is where "the benchmarks looked fine but the model was weirdly bad on Tuesday afternoons" gets diagnosed *before* the model is ramped to users. It is also where the most common statistical mistake in production eval — treating production traffic as if it were an offline dataset and slapping a t-test on the difference of means — turns a shadow into a source of misleading confidence. This chapter is about the mechanics of running the shadow correctly and about reading the result correctly under traffic that is drifting, correlated, and heavy-tailed.

## Shadow versus dark launch versus mirror: three names, three shapes

The vocabulary is not consistent across the industry, so it is worth being precise. In this module:

- **Shadow deployment.** For every incoming production request, the same request is dispatched in parallel to both the incumbent and the candidate. The incumbent's response is returned to the user; the candidate's response is captured, evaluated, and discarded. Users see nothing new. The comparison is fully *paired* — every request has both an incumbent response and a candidate response — which is the statistically most powerful shape for comparison because it removes between-request variance from the estimator (mod-101 Chapter 3's paired-sample discipline). Cost: you pay for full candidate inference on the shadowed fraction of traffic, doubling that fraction's serving cost.
- **Dark launch.** The candidate is deployed to production infrastructure and receives a fraction of traffic (typically a small percentage from a fixed cohort or from an internal-users-only cohort), and its output *is* returned to those users, but the feature or model change is not announced or attributed. Dark launch is a shape of A/B experiment (Chapter 4), not a shape of shadow — it collects real user response signals but at the cost of exposing users to the candidate.
- **Mirror / traffic replay.** The candidate is fed a recorded log of past production requests (a mirror), replayed offline. This is close to a shadow but with no live coupling — the incumbent's response is whatever was recorded at the time of the original request, and the candidate is evaluated against a snapshot of a distribution that is now stale.

The three sit on a "how live is the comparison?" axis. Mirror is fully offline. Shadow is live but user-invisible. Dark launch is live and user-visible. Each has a legitimate use; a common launch discipline uses all three in sequence:

1. Offline regression suite (Chapter 2) → mirror replay against last month's traffic → shadow on live traffic → dark launch (small A/B, Chapter 4) → full A/B → ramp.

Chapter 3 is about the shadow. Chapter 4 picks up at dark-launch / A/B.

## Why shadow beats mirror when you can afford it

Mirror replay is cheap because the candidate does not consume live compute at request time — you replay from a log. Its downside is that the log is stale, and *how* stale matters more than it seems.

- **Distribution shift over days and weeks.** Product surface changes, marketing campaigns, competitor outages, seasonal patterns (education traffic collapses in July; retail traffic spikes in November) all move the request distribution enough that a candidate that looked great on last month's replay can be materially different on this week's traffic.
- **Coupling to the incumbent's behavior.** If the incumbent has been serving the user for a session already, the user's next request is a function of the incumbent's previous responses. The candidate is being asked to answer a question that was shaped by a *different* model's prior turn. A replay from the incumbent's log is *definitionally* out-of-distribution for the candidate on multi-turn conversations.
- **Feedback loops on retrieval / tools.** If the product does retrieval or tool-calling, the corpus and tool responses may have moved between the log capture and the replay — a URL that was 200 OK in June is now a 404, an inventory API returns different SKUs, a search index has been re-embedded.

Shadow avoids all three because the request being processed is *this* request from *this* user in *this* moment, with the current retrieval corpus and the current tool state. What shadow does not avoid: the multi-turn coupling to the incumbent's prior turns is still present on the candidate side. If the previous turn in the session was served by the incumbent, the candidate is answering a question shaped by the incumbent's response. Chapter 4's A/B discusses session-level randomization as the fix; the shadow deployment discipline acknowledges the limitation and reports its effect explicitly.

## Two failure modes of the setup itself

Before any statistics, the shadow's plumbing has to be correct. Two failure modes are common and both invalidate everything downstream.

### Failure mode: side-effects leak from the candidate

The candidate is supposed to be user-invisible, but a shadow that calls tools (a search API, a code executor, an email-sending function, a database write) is not invisible to the *world*. If the candidate's shadow call sends an email or writes to a database, the user experiences the effect of two model runs on their session, one of which they did not consent to.

The mitigation is a **shadow-mode adapter layer** in the model runtime: every tool the candidate calls must be either (a) idempotent and read-only (fine to call twice), or (b) intercepted and replaced with a mock that returns a plausible response for evaluation purposes without executing the side effect. The list of "which tools are shadow-safe" is a maintained artifact, and adding a new tool to the product requires a shadow-safety review. Skipping this is the standard way to accidentally double-book calendar events during a shadow rollout.

### Failure mode: the shadow runs on the incumbent's already-processed context

If the candidate is fed *the input the incumbent saw* it must be fed it *before* the incumbent has done anything to shared state (system prompt injections, memory writes, RAG corpus updates from prior tool calls). If the shadow receives its input after the incumbent has finished its full turn, the candidate is answering a modified version of the problem. Solution: the shadow dispatch happens at the same point in the request pipeline as the incumbent dispatch — same input snapshot, same retrieval results, same tool responses fed to both.

Symmetry check: for any observable that both models see (system prompt version, retrieval snippets, tool outputs), the values are byte-identical between the incumbent's and the candidate's inputs. Any divergence is a plumbing bug and invalidates the paired comparison.

## What metrics the shadow can actually compute

A shadow deployment captures `(request, incumbent_response, candidate_response)` triples. What it does *not* capture is a user reaction to the candidate — nobody saw the candidate's output. This constrains the metric set:

- **Judge-graded quality metrics** (helpfulness, correctness, tone, format compliance) can be computed on both responses using the mod-105 judge stack. These are the workhorses of shadow reporting.
- **Text-similarity metrics** between the two responses (edit distance, embedding similarity, output-format match rate, refusal-agreement rate) quantify *how different* the candidate is from the incumbent, not whether it is better. These are useful for anomaly detection ("the candidate produced meaningfully different responses on 40% of traffic — investigate before A/B") but they are not a quality signal.
- **Safety-classifier outputs** on both responses (refusal detected, toxicity score, PII leak flag) can be compared as paired proportions.
- **Serving metrics** on the shadow side (TTFT, TPOT, total latency, tokens generated) — the candidate's serving characteristics on the actual request distribution, which is often more informative than a fixed-prompt benchmark. Chapter 7 discusses the interpretation.

What the shadow **cannot** compute:

- Any signal that requires a user seeing the output (thumbs-up rate, session-continuation rate, downstream conversion, revenue-per-request). Those are dark-launch / A/B metrics (Chapter 4).
- Any signal that requires the candidate's response to affect subsequent turns (multi-turn task completion rate, agent success rate on multi-step tasks). The candidate's response was thrown away, so the next turn was served by the incumbent, so the "candidate's success on the multi-turn task" is not measurable without exposing users.

The correct discipline is to explicitly enumerate what the shadow does and does not measure in the shadow report, and to name the metrics that require an A/B for later. A shadow report that presents a "quality delta" without disclaiming what it cannot measure is over-claiming.

## Reading the result correctly under non-IID traffic

Production traffic violates three assumptions that the naïve "compute the mean, run a t-test" pipeline depends on. Each one has a specific fix.

### Violation 1: independence — requests within a user's session are correlated

If a user asks five questions in a session and all five are captured by the shadow, those five `(inc, cand)` pairs are not five independent draws from the request distribution — they share the user, the topic, the language, the domain, and the mood. Treating them as five independent observations shrinks the standard error to a fraction of its true value; the confidence interval on your delta will look tighter than the reality, and you will call differences significant that are not.

The fix is **cluster-robust standard errors** or a **cluster bootstrap**: the unit of resampling is the session (or the user), not the individual request. In practice: when bootstrapping over the shadow-collected data, resample sessions with replacement and take *all* requests in the resampled session. The confidence interval widens, sometimes a lot; that widening is real information about how many independent signals you actually collected.

Recipe (paired shadow, per-session cluster bootstrap):

```python
def paired_delta_ci(paired_samples, cluster_key, n_boot=2000, seed=17):
    # paired_samples: list of {"cluster": user_or_session_id, "metric_inc": float, "metric_cand": float}
    import numpy as np
    rng = np.random.default_rng(seed)
    clusters = {}
    for s in paired_samples:
        clusters.setdefault(s[cluster_key], []).append(s)
    cluster_ids = list(clusters.keys())
    deltas = []
    for _ in range(n_boot):
        sampled = rng.choice(cluster_ids, size=len(cluster_ids), replace=True)
        chunks = [clusters[c] for c in sampled]
        flat = [s for chunk in chunks for s in chunk]
        deltas.append(
            np.mean([s["metric_cand"] - s["metric_inc"] for s in flat])
        )
    return float(np.mean(deltas)), (
        float(np.percentile(deltas, 2.5)),
        float(np.percentile(deltas, 97.5)),
    )
```

The point is not the code. The point is: **the cluster is the unit of independence, not the request.** Every shadow report should state its cluster unit and use it in the CI.

### Violation 2: identical distribution — traffic drifts over the observation window

A shadow that runs for a week is observing a distribution that changes over that week: weekend versus weekday mix, morning versus evening mix, marketing-campaign spikes, seasonal effects. If the candidate is subtly worse on the segment that dominates weekend traffic and subtly better on the segment that dominates weekday traffic, the overall delta averages the two — and the sign of the reported delta depends on how many weekend days versus weekday days fell in your window.

Two disciplines address this:

- **Stratify the analysis by the axis you suspect matters.** Compute the delta separately for weekday-morning, weekday-evening, weekend, mobile, desktop, English, non-English, session-length bucket. If the deltas are consistent in sign and magnitude across strata, the aggregate delta is trustworthy. If some strata are meaningfully worse and others are meaningfully better, the aggregate hides a real segment-level regression that will bite when A/B ramps to a cohort that happens to over-index on the regressing segment.
- **Report the window explicitly.** A shadow report for a 7-day window covers a specific set of days with a specific traffic composition. The header of the report names the window (`2026-07-15 09:00 UTC → 2026-07-22 09:00 UTC`) and, ideally, the top-line composition (`68% weekday, 32% weekend; 51% mobile; 88% English`). The comparison across two shadows requires similar windows or an explicit re-weighting.

For high-stakes releases, the shadow runs for a minimum of one full week to cover the weekly seasonality, and preferably long enough to include one full weekend and one full week-day-with-marketing.

### Violation 3: heavy tails — a small fraction of requests dominates the metric

Judge scores are bounded, but token counts, latencies, and cost-per-request are heavy-tailed: a small fraction of requests (long-context prompts, agentic multi-step tasks, users who paste enormous documents) contribute a disproportionate share of the sum. A candidate that is 5% slower on the median request but 40% slower on the 99th-percentile request will show a small delta in the mean and a large delta in the P95 — and the P95 is often what the SLA cares about.

The disciplines here are: **report the full distribution, not the mean.** Every heavy-tailed metric should be reported as `mean [p50, p90, p95, p99]` for both incumbent and candidate, plus the delta at each percentile. The P95 delta and the mean delta can point in different directions; when they do, the launch review needs to see both. mod-103's operating-point discipline applies: the percentile you care about is a product decision, not an eval decision.

## The reporting shape

A defensible shadow report has the following shape (values illustrative, cluster unit = `session_id`):

```
# Shadow Report — release-candidate abc123 vs incumbent prod-v0.14.1

Window: 2026-07-15 09:00 UTC → 2026-07-22 09:00 UTC (7 days)
Traffic sampled: 12.0% of live traffic, side-effect-suppressed
Total requests: 1,842,306 across 386,414 sessions
Cluster unit for CIs: session_id
Judge:  internal-judge-v1.2 (prompt SHA 33bb44cc, backend claude-3-7-sonnet-20250219)

## Quality metrics (paired, judge-graded, cluster-bootstrap 95% CIs)
| metric                        | incumbent | candidate |     Δ | 95% CI          |
|-------------------------------|-----------|-----------|-------|-----------------|
| helpfulness (0–5)             |     4.12  |     4.15  | +0.03 | [+0.01, +0.05]  |
| correctness (0/1)             |     0.86  |     0.88  | +0.02 | [+0.00, +0.03]  |
| refusal (0/1)                 |     0.06  |     0.05  | -0.01 | [-0.02, -0.00]  |

## Safety-classifier metrics
| metric                        | incumbent | candidate |     Δ | 95% CI          |
|-------------------------------|-----------|-----------|-------|-----------------|
| toxicity flag rate            |    0.002  |    0.001  | -0.001| [-0.002, +0.000]|
| PII-leak flag rate            |    0.0005 |    0.0004 | -0.0001| [-0.0004, +0.0002]|

## Serving metrics (heavy-tailed — full distribution)
| metric        | inc mean | cand mean |  inc p95 | cand p95 |  ΔP95 |  ΔP99  |
|---------------|----------|-----------|----------|----------|-------|--------|
| TTFT (ms)     |     412  |      438  |     1240 |     1385 |  +145 |  +290  |
| completion(ms)|     830  |      814  |     2100 |     2010 |   -90 |  -110  |
| total tokens  |    186   |     201   |     640  |     705  |   +65 |  +140  |

## Stratified deltas (helpfulness)
| stratum                | n_sess |     Δ | 95% CI          |
|------------------------|--------|-------|-----------------|
| weekday morning        |  95k   | +0.04 | [+0.02, +0.06]  |
| weekday evening        |  84k   | +0.03 | [+0.01, +0.05]  |
| weekend                | 128k   | +0.03 | [+0.01, +0.05]  |
| mobile                 | 197k   | +0.02 | [+0.01, +0.04]  |
| desktop                | 189k   | +0.04 | [+0.02, +0.06]  |
| non-English            |  46k   | -0.02 | [-0.05, +0.01]  |

## Divergence indicators
| indicator                                   |    value  |
|---------------------------------------------|-----------|
| responses with edit distance > 100          |    41.2%  |
| refusal agreement rate (both refuse or both help) | 96.8% |
| refusal-flipped (inc refuse, cand help)     |     1.9%  |
| refusal-flipped (cand refuse, inc help)     |     1.3%  |

## Not measurable from shadow
- Downstream conversion, session-continuation, thumbs-up rate: require A/B.
- Multi-turn task completion where candidate response affects subsequent turns:
  requires session-level A/B (Chapter 4).
```

The stratified deltas are the load-bearing part of the report. The aggregate "+0.03 helpfulness" reads well; the non-English stratum's "-0.02 [-0.05, +0.01]" is the warning that the launch review needs to see before deciding whether the aggregate delta is trustworthy.

## Interpreting the result

A shadow gives one of three verdicts:

- **Green.** Aggregate delta is favorable, stratified deltas are consistent in sign, no stratum has a large adverse delta with a tight CI, serving metrics are within the SLA cushion. Promote to A/B.
- **Yellow.** Aggregate is favorable but some strata show adverse deltas (as in the non-English row above), or serving metrics have moved unfavorably at the P95 or P99, or divergence indicators show that the candidate is producing very different outputs from the incumbent. Root-cause the yellow before promoting. This is where the shadow earns its cost — catching segment-level regressions or unexpected divergence before real users see them.
- **Red.** Aggregate delta is unfavorable, or a safety-classifier metric has regressed with a tight CI, or a serving metric has blown a hard SLO. Do not promote. Return the candidate for iteration.

The threshold for green versus yellow is a team decision, and it should be pre-registered along with the shadow plan — "we promote if aggregate helpfulness delta is ≥ 0 with a CI whose lower bound is ≥ -0.01 and no stratum shows a delta ≤ -0.02 with a CI whose upper bound is ≤ 0." Writing the rule down before running the shadow is the mod-101 discipline that keeps the interpretation honest.

## Guidance for the eval author

- **Shadow-safe every tool.** If a tool has side effects, it needs a mock in shadow mode. No exceptions.
- **Symmetry between incumbent and candidate inputs.** Any divergence in system prompt, retrieval snippets, or tool outputs between the two invalidates the paired comparison.
- **Cluster-robust CIs, always.** Sessions and users are the units of independence, not individual requests. Cluster-bootstrap by session at minimum, by user for stronger correlation cases.
- **Stratify.** The overall delta hides segment-level regressions. Weekday/weekend, mobile/desktop, English/non-English, session-length, and any product-specific cohort you know matters.
- **Percentiles for heavy-tailed metrics.** TTFT mean is nearly useless; P95 and P99 are what the SLA lives on.
- **Divergence indicators are for anomaly detection, not for quality.** "The two models disagree on 40% of responses" is a red flag to investigate, not a quality delta.
- **Pre-register the decision rule.** Green/yellow/red thresholds are written before the shadow starts, not after the numbers land.
- **Disclose what the shadow cannot measure.** Downstream user response, multi-turn completion, revenue-per-request all wait for A/B; the shadow report says so.
- **Run for a full seasonality period.** Shorter windows over-fit to whatever composition of days happened to fall in the window.

## Summary

A shadow deployment dispatches production traffic to both the incumbent and the candidate in parallel, returns only the incumbent's response to the user, and captures paired `(incumbent, candidate)` observations for evaluation. It is the strongest available comparison short of exposing users, because it runs on the true request distribution — but only when the plumbing enforces side-effect suppression and input symmetry, and only when the statistics respect that production traffic is clustered by session, drifting over the window, and heavy-tailed. The reporting shape is stratified deltas with cluster-robust CIs, distributions rather than means for latency and cost, and a pre-registered green/yellow/red rule that promotes candidates to A/B only when both the aggregate and the strata support the decision. The next chapter takes the candidate that has passed shadow and puts it into a controlled A/B with CUPED variance reduction, session-level randomization, and the pre-registered hypotheses that let the launch review reach a defensible ramp/hold/rollback decision.
