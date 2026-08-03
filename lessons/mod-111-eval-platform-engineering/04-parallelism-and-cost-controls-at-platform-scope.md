# Parallelism and Cost Controls at Platform Scope

An eval platform's cost curve is not a linear function of the number of evals it runs. It bends sharply upward at the point where every team can freely fan out concurrent requests to the same vendor, and it bends sharply downward at the point where the platform's controls are strict enough that the marginal team's marginal eval no longer costs the shared budget an unpredictable amount. Getting the second curve is what makes the platform sustainable; failing at it is how eval platforms get shut down after a "surprise" seven-figure vendor bill.

This chapter is about the specific control surfaces that make the second curve possible: token budgets that are enforced per-run and per-tenant; runner budgets that cap concurrent worker allocation; judge-tier routing that spends the expensive judge only where it is worth spending; priority queues that route release-blocking traffic ahead of research sweeps; and rate-limit shaping that keeps the platform courteous to its vendors and its self-hosted serving stack alike.

## The four cost axes

Every eval run consumes resources along at least four axes; the platform's control surface has to reason about each independently.

- **Token spend.** The dominant cost for hosted-vendor calls and the main proxy for cost even on self-hosted stacks (where "tokens generated" is a good stand-in for "GPU-seconds occupied"). Denominated in dollars for hosted, in accelerator-hours for self-hosted, and increasingly denominated separately for input-tokens, output-tokens, and cached-input-tokens (see the vendor caching discussion below).
- **Wall-clock time.** A release-blocking eval that takes 45 minutes and a research sweep that takes 12 hours are the same "eval" to Chapter 3's orchestration surface but need very different queueing treatment. Wall-clock is what latency SLOs are denominated in (Chapter 6).
- **Concurrency slots.** The number of simultaneous in-flight requests the platform is willing to submit to a given backend. Vendors publish rate limits per API key; self-hosted serving stacks have a batch-size envelope beyond which throughput does not scale. The platform allocates from this pool.
- **Human review budget.** The number of human labels an eval needs (mod-106 covers the labor side). A judge-graded eval that expects a 5% human-review sample has a downstream labor cost that the platform's budget accounting has to track.

A control surface that only meters tokens gets caught out the first time an unlucky sweep occupies 90% of the vendor rate-limit pool for six hours; a surface that only meters concurrency gets caught out the first time a fan-out job burns half a million dollars because "the concurrency was fine."

## Token budgets: the first-order control

Every run submitted through Chapter 3's plane declares a token budget. The budget is enforced at three levels:

- **Per-run budget.** Declared at request time, in tokens (usually split by input, output, and — where the backend supports prompt caching — cached-input). The platform decrements the run's budget as each item's usage is reported and terminates the run at a configurable threshold (typically 100% for hard-blocking, 120% with a warning for soft budgets).
- **Per-tenant budget.** A rolling window budget (per-day, per-week, per-quarter) for each requesting team or product. When a new run's projected budget would exceed the tenant's remaining allowance, the plane either queues the run until the window rolls or rejects it with a clear error. Tenant budgets are set by the finance/platform-owner conversation, not by individual eval authors.
- **Platform-wide budget.** The aggregate ceiling for the platform's vendor spend over a period. Approaches to this ceiling trigger the platform's cost alerts (Chapter 6) and, at hard thresholds, throttle low-priority workloads.

Two design properties keep the token-budget mechanism honest:

- **Budgets are pessimistically pre-allocated, not just post-hoc metered.** When a run is queued, its projected budget is subtracted from the tenant's remaining allowance immediately. If the run finishes under budget, the difference is returned; if it exhausts, no harm done. Purely post-hoc accounting is what lets a burst of concurrent runs collectively blow through a tenant's daily budget because none of them individually knew about the others.
- **Projections use the task's historical distribution.** A run's projection is not "the requesting user's estimate" (which is optimistic) but the task's own historical mean-plus-margin (from Chapter 5's warehouse). If the same task has consistently used 250K tokens on this model family, a request that declares a 100K budget is either rejected or asked to raise its budget explicitly.

## Judge-tier routing: the multi-tier judge economy

LLM-as-judge is where a large fraction of platform token spend actually lands. A judge-graded run on a 5K-item benchmark with a mid-tier judge (say, Claude Sonnet or GPT-4-class) can easily cost more than the run of the model-under-test itself. Judge-tier routing is the platform's answer.

The idea is to define a small set of tiers with different cost/agreement profiles and to route each judged item to the cheapest tier that meets the accuracy requirement for the item's use case:

- **Tier 0 — classical / deterministic scorers.** BLEU, ROUGE, exact match, F1, regex classifiers. Effectively free. Used for the metrics where an exact-match answer key exists.
- **Tier 1 — small dedicated judge.** A fine-tuned small-scale judge model (Prometheus 2, an internal rubric-trained judge, or a mid-size hosted judge). Cheap per call, calibrated against humans on a documented subset (Chapter 2's registry stores this calibration).
- **Tier 2 — mid-tier frontier judge.** A hosted mid-tier model (Claude Haiku-class, GPT-4o-mini-class). More expensive; usually the default for release-blocking judged runs.
- **Tier 3 — top-tier frontier judge.** The most capable hosted judge (Claude Opus, GPT-4-class). Reserved for high-stakes items — safety adjudication, contested rubric decisions, disagreement resolution.

Routing policies that work in practice:

- **Route by task registration.** The registered task nominates a default judge tier. Release-blocking safety tasks land in Tier 3; a large-scale capability sweep lands in Tier 1 or 2. This is the primary lever and requires no per-item decision logic.
- **Route by disagreement escalation.** For items where two Tier 1 judges disagree, escalate to Tier 2 or 3. This composes with mod-105's panel-of-judges pattern and Chapter 2's judge registration (the panel is a single judge revision that composes lower-tier judges internally).
- **Route by confidence.** For judges that emit a confidence score, escalate low-confidence items to a higher tier. Requires calibration work upstream to make the confidence numbers meaningful.
- **Route by cost budget remaining.** Under budget pressure, downgrade routing (Tier 3 → Tier 2 for the tail of items). This is the pressure-relief valve; the platform logs when it triggers so that repeated triggering surfaces as a signal that either the budget is too tight or the tier defaults are misconfigured.

The routing policy is a declared property of the task, not an ad-hoc runtime decision. Chapter 5's warehouse records the tier that actually judged each item, so "was this item graded at Tier 3 or downgraded to Tier 2?" is a query.

## Runner budgets: bounding concurrency and worker allocation

Chapter 3's plane dispatches runs to runner-adapter pools. Each pool has a bounded concurrency — a fixed number of workers or a fixed serving-stack throughput — and the plane must not oversubscribe.

Runner budgets have three shapes worth naming:

- **Per-pool concurrency ceiling.** The maximum number of simultaneous runs the plane will dispatch to a given adapter pool. Set from the pool's actual throughput measurements: if lm-eval-harness workers on your infrastructure can steadily execute six concurrent MMLU sweeps before the model-serving backend starts thrashing, the ceiling is six.
- **Per-model backend concurrency ceiling.** The maximum number of simultaneous requests the plane will submit to a given model backend. Distinct from per-pool because two runner pools may share a backend; concurrency limits apply to the backend, not the runner.
- **Priority classes with reserved slots.** Some fraction of concurrency (typically 20–40%) is reserved for release-blocking runs. When the ceiling is exhausted by lower-priority sweeps, a release-blocking run preempts a slot rather than waiting.

The observed failure mode without these controls is not that the platform crashes — it is that everyone's latency SLO drifts silently, because a single 12-hour research sweep occupies every worker and the release-blocking run that arrived an hour into the sweep just waits.

## Priority queues and workload classes

The plane's queue is not FIFO. Different eval workloads have different urgency and different resource profiles; the platform classifies them and applies different queueing rules per class.

A workable set of classes:

- **`release-blocking`**: Fixed sample size, tight wall-clock SLO (typically ≤ 60 min P95), highest queue priority, reserved concurrency slots, exempt from vendor-throttle downgrades. Usually the offline regression suite from mod-110 Chapter 2.
- **`safety-canary`**: Similar to release-blocking but often longer-running (adversarial pass, red-team suite from mod-109); may run overnight but has a hard deadline (before the morning launch review). Reserved concurrency, exempt from downgrades.
- **`research-sweep`**: Best-effort. Runs when there is capacity; can be preempted by higher-priority classes; typically the largest token consumer on the platform.
- **`interactive`**: One-off runs from a human user (an engineer debugging a judge, an analyst investigating a slice). Low latency SLO, low token budget per run, best-effort concurrency.
- **`scheduled-refresh`**: Recurring baseline runs (Chapter 6's re-baseline schedule). Wall-clock-flexible but must complete within a documented window (weekly, monthly).

Each class has documented SLOs, documented budget defaults, and documented preemption rules. Users request a class at submission time (defaulting to `interactive` if unspecified); Chapter 6 wires the CI system to submit as `release-blocking` explicitly.

The queueing implementation is deliberately not a subject of this chapter — Kubernetes queues, Argo, Prefect, an internal work-stealing scheduler — any of them can implement the priority mechanics above. What matters is that the classes are named, the SLOs are declared, and the preemption rules are policy, not accident.

## Rate-limit shaping and backoff

The platform is downstream of one or more model backends, each with its own rate limits. Vendor rate limits are typically declared as requests-per-minute (RPM) and tokens-per-minute (TPM) per API key, with separate ceilings for input and output tokens on some vendors. Self-hosted serving stacks have their own effective ceilings governed by batch-size and memory pressure.

The platform's model-client layer (Chapter 3) implements a shared rate-limit shaper:

- **Token-bucket accounting per backend.** The client maintains a running consumption estimate and paces its own requests to stay under the declared limit — usually at 80–90% of the ceiling to leave headroom for burstiness. Exceeding the ceiling produces vendor 429s that are more expensive to handle than pacing correctly in the first place.
- **Exponential backoff on 429 and 5xx.** Retries respect a jittered exponential backoff (starting at ~500ms, doubling to a ~30s cap) rather than tight-loop retrying, and are attributed to the same run's budget (retry tokens are still tokens).
- **Backpressure propagation to the queue.** When a backend's error rate spikes, the plane slows queue dispatch for runs that target that backend rather than firing them all at the wall. Chapter 6's SLOs distinguish "eval failed because of platform bug" from "eval slowed because of upstream backpressure."
- **Retry accounting.** Every retry emits a lineage event; a run whose completion required 12 retries is a different quality of measurement than a run that completed on the first try, and Chapter 5's warehouse retains that fact.

Vendor-side changes to the rate-limit envelope (a limit raise, a new model tier with a different envelope) are platform-level events, not per-run events. The shaper's parameters are configuration, and configuration changes go through the same review as any other platform change.

## Prompt caching and other structural cost reductions

Modern hosted vendors expose input-side prompt caching: a prompt prefix that is reused across many calls (a rubric, a set of few-shot examples, a long system message) is billed at a discounted rate on subsequent calls that share the prefix. Anthropic's prompt caching, OpenAI's prompt caching, and Google's context caching each have their own mechanics but the shape is the same: identify a stable prefix, mark it as cacheable, save 50–90% of the input-token cost on the cached portion.

For eval workloads, prompt caching is particularly high-leverage:

- **Judge rubrics are extremely cacheable.** A rubric prompt is typically 1–4K tokens and is used identically across every item in a run. Caching cuts the judge input cost by an order of magnitude.
- **Few-shot task templates are cacheable.** The few-shot preamble to a benchmark item is stable across items in the run; caching it drops the per-item input cost dramatically.
- **Retrieval contexts are usually not cacheable.** RAG's retrieved context changes per query and is the hardest to cache. This is a good thing to know when designing budgets — RAG judge runs cost more per item than closed-book judge runs, even at the same token count.

The platform's model-client layer exposes prompt-caching hints (the vendor-specific mechanism is behind the interface); the runner adapters that consume it (particularly the judge adapters) opt in. Chapter 5 records cache-hit rates on every call so that budget projections can distinguish cache-warm and cache-cold runs.

Batch APIs (Anthropic's Message Batches, OpenAI's Batch API) are a related lever: overnight batch pricing is typically half the synchronous price, at the cost of a 24-hour SLA on completion. Batch is appropriate for `research-sweep` and `scheduled-refresh` classes; it is not appropriate for `release-blocking` (the SLA doesn't fit). The platform exposes batch as an option and the workload class's defaults determine whether it's used.

## Cost attribution and the finance conversation

Once the platform is metering all four axes, the natural downstream product is per-tenant cost attribution — a monthly report of what each team's evals actually cost the platform.

Two properties of the attribution matter:

- **The unit is the run, not the API call.** Individual API calls are aggregated up to their originating eval run; runs are aggregated up to their requesting team and release-candidate. This is the granularity a finance conversation actually operates at.
- **Attribution is joinable to outcomes.** Chapter 5's warehouse lets you ask "what did the safety-eval program cost this quarter, and what was its ratio of block-decisions to allow-decisions?" without a manual data-pull. The finance conversation is much healthier when it can compare cost to signal produced, and much unhealthier when it can only compare cost to volume produced.

The attribution mechanism is what turns "the eval platform costs a lot" from a vague complaint into a decision — invest in cheaper judge tiers, prune low-signal evals, raise or lower per-tenant budgets. Without it, the same complaint is a periodic anxiety with no next step.

## Failure modes these controls are designed to prevent

### Failure mode: the surprise seven-figure vendor bill

A team fans out a research sweep at 03:00 with a `for model in models: for task in tasks: subprocess.run(...)` pattern that has no budget declared. Twelve hours later the vendor has invoiced a quarter of the year's budget, the finance team has escalated to the CTO, and every other team's evals for the next month are throttled or paused.

Platform mitigation: pessimistic budget pre-allocation, per-tenant budgets, projections from historical distribution, hard token ceilings on individual runs. The fan-out job hits the ceiling long before the finance team ever sees the number.

### Failure mode: the release-blocking run stuck behind an 8-hour sweep

A release-blocking eval is submitted at 10:00; a research sweep started at 09:00 is occupying every worker; the release-blocking run doesn't start until 17:00; the launch review has already moved without it.

Platform mitigation: priority classes with reserved concurrency slots, preemption of lower-priority runs, wall-clock SLOs per class. Release-blocking runs get their slot at 10:00 regardless of what else is running.

### Failure mode: the vendor cuts the platform off mid-day

A burst of concurrent runs pushes the platform above the vendor's declared limit; the vendor 429s aggressively; the platform's naive retry logic amplifies the 429 rate; eventually the vendor rate-limits the platform's account entirely and the release-blocking runs fail.

Platform mitigation: token-bucket accounting under-provisioned to 80% of ceiling, jittered exponential backoff, backpressure into the queue rather than the wall. The platform stays courteous to the vendor even under load.

### Failure mode: the judge is at Tier 3 for a research sweep and Tier 1 for the release

The default tier assignment is inverted somewhere. A research sweep is paying frontier-judge rates for cheap items; a release-blocking eval is grading with a mid-tier judge that is not calibrated for its use case. The signal-per-dollar ratio is inside-out.

Platform mitigation: tier defaults declared at task registration; the warehouse reports on tier-by-tenant so that inversions surface. The registry's judge revisions carry their calibration data (Chapter 2), which lets the platform surface "this judge is not calibrated for this task" as a lint warning.

## Guidance for the platform engineer

- **Meter all four axes.** Tokens, wall-clock, concurrency slots, and human review budget. A control surface that only meters tokens misses half the problems.
- **Pre-allocate, don't post-account.** Optimistic post-hoc metering makes it structurally impossible to bound the burst-cost of concurrent runs.
- **Tier judges by task, not by heroism.** Route by registered task's tier default; escalate on disagreement or low confidence. Ad-hoc per-item routing decisions are a slippery slope.
- **Reserve concurrency for release-blocking.** The queue is not FIFO. Reserved slots for the smallest, most urgent class prevent a 12-hour sweep from silently blowing a release deadline.
- **Under-provision to the vendor ceiling.** 80% of the vendor's declared limit is the shaper's target. The remaining 20% is your defense against burst and vendor-side variance.
- **Enable prompt caching by default for judge rubrics.** The single highest-leverage cost reduction available; opt-in is the wrong default.
- **Attribute cost to runs, not to API calls.** Finance conversations happen at the run level. Aggregate up before you present.

## Summary

Platform-scope cost and parallelism controls are what let an eval platform serve many teams' evals without the marginal team's marginal eval blowing up the shared budget. The four cost axes — token spend, wall-clock, concurrency slots, and human review — each require their own control surface: per-run and per-tenant token budgets with pessimistic pre-allocation and projections drawn from the warehouse; judge-tier routing that spends the expensive judge only where it's worth spending; runner budgets that bound concurrency per adapter and per model backend; priority queues with named workload classes; and rate-limit shaping that keeps the platform under 80% of vendor ceilings. Prompt caching and batch APIs are structural cost reductions that pay for themselves quickly on judge-heavy workloads. Cost attribution at the run level (not the API-call level) is what turns "eval is expensive" into a decidable finance conversation. The next chapter is about what happens after the run completes — the warehouse where every result, with its full artifact-hash lineage, becomes queryable.
