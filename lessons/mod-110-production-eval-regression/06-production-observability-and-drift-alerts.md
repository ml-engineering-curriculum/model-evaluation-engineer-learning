# Production Observability and Drift Alerts

The four altitudes covered so far — offline regression, shadow, A/B, sequential monitoring — each produce data. The offline suite emits gate reports. The shadow emits paired comparisons. The A/B emits primary-metric analyses. The continuous monitors emit confidence sequences on live streams. Without a platform that ingests, stores, indexes, and alerts on that data, all of it lives in scattered notebooks, one-off dashboards, and the tribal knowledge of whoever ran the last analysis. A month later nobody can reproduce the shadow report from July, the monitor that fired on Tuesday cannot be root-caused because the traces have rotated out of the retention window, and the launch review has to accept a screenshot as evidence.

This chapter walks the three widely-adopted open-source LLM observability platforms — **Arize Phoenix**, **Langfuse**, and **W&B Weave** — and the drift-alert designs that make them useful. The goal is not a vendor comparison; it is a common vocabulary for what an LLM eval platform must provide, why the eval discipline from this module needs each piece, and how to design drift and judge-drift alerts on top.

## What an LLM observability platform actually stores

Regardless of vendor, a modern LLM observability platform is built around a small number of primitives that map directly onto the OpenTelemetry / OpenInference tracing model:

- **Traces and spans.** A trace is a single user-facing operation (e.g., a single chat turn or a single agent invocation). A trace contains one or more spans, each representing a sub-operation — an LLM call, a tool call, a retrieval, a rerank. Spans are nested; the root span is the top-level user request.
- **LLM call spans.** Structured records of a call to a model: input messages, output message, model name, temperature, prompt tokens, completion tokens, latency, cost estimate, arbitrary metadata (session id, user id, product surface, request id).
- **Retrieval spans.** For RAG systems: the query, the retrieved documents, similarity scores, retrieval latency.
- **Tool-call spans.** Tool name, tool arguments (structured), tool response, latency.
- **Evaluations.** Records attached to a span (usually the LLM call span or the trace root) containing a metric name, a value, an evaluator name, a rationale, and metadata. The `(trace_id, span_id, metric_name, evaluator_name) → value` tuple is the atom of an eval-on-observability workflow.
- **Datasets.** Named, versioned collections of examples used for offline eval. A dataset row is a `(input, expected_output, metadata)` triple. The platform typically supports running an evaluator across a dataset and recording the resulting evaluation rows.

The OpenInference spec (an OpenTelemetry-family convention maintained by Arize) is the current de facto standard for what fields go on an LLM span; both Phoenix and (increasingly) Langfuse can ingest OpenInference-conformant OTel traces. W&B Weave uses its own trace schema but exposes similar primitives.

The consequence for the eval discipline: **every metric this module has produced — a shadow's paired judge score, an A/B's primary metric per exposure, a continuous monitor's per-request refusal flag — can be attached to a span as an evaluation.** The platform becomes the join point where "this metric" and "this trace" and "this user session" and "this product surface" are all queryable together. Without the platform, those joins have to be reconstructed by hand from three different logging systems.

## The three reference platforms, briefly

The following are all open-source (with a commercial hosted option), can be self-hosted, and share the traces-plus-evaluations model. Feature sets converge over time; treat this as the shape of the space at the time of writing rather than a permanent taxonomy.

### Arize Phoenix

Arize's open-source LLM observability tool. Distinguishing features from the eval-integration angle:

- **Deep OpenInference integration.** Phoenix is Arize's reference implementation of the OpenInference spec; instrumentation SDKs (`openinference-instrumentation-*`) for LangChain, LlamaIndex, OpenAI SDK, LiteLLM, and others emit spans directly to Phoenix.
- **Evaluator framework.** Phoenix ships with a library of pre-built LLM-as-judge evaluators (hallucination, relevance, toxicity, correctness against reference) and a `phoenix.experiments` API for running them across a dataset.
- **Drift monitoring.** Phoenix's original heritage is the ML monitoring product (embeddings drift, prediction drift for classical ML); the LLM side has a UMAP-projected embedding visualization of production traces vs. a reference dataset and per-attribute drift statistics.

### Langfuse

Open-source LLM observability with a Postgres-backed data plane and a strong dashboard / analytics layer. Distinguishing features:

- **Trace-first UI.** Langfuse's UI is oriented around trace exploration — a trace's spans, their inputs/outputs, their scores, and their user/session context are the primary view.
- **Score model with types.** Langfuse's `Score` primitive is typed (`NUMERIC`, `BOOLEAN`, `CATEGORICAL`) and can be attached to a trace, an observation (span), or a session. Scores can be authored by a human (annotation queue), by an evaluator (SDK), or by another API caller.
- **Prompt management.** First-class versioned prompt library with A/B and rollout support. Ties directly into the deployment discipline this module covers — the prompt for the evaluator is itself a versioned artifact whose changes should trigger re-baselining (see the judge-drift discussion below).

### Weights & Biases Weave

W&B's LLM observability layer, integrated with the broader W&B experiment-tracking product. Distinguishing features:

- **Deep integration with W&B experiment tracking.** For teams already using W&B for model-training experiments, Weave sits in the same account and shares projects, runs, and reports.
- **`weave.op` decorator model.** Any Python function annotated with `@weave.op()` is traced automatically — inputs, outputs, and metadata are captured on call. This makes instrumentation particularly lightweight for research-style code.
- **Evaluation reports.** W&B's reporting layer supports named eval runs across datasets with side-by-side comparison of model versions on the same eval.

All three platforms can be the storage substrate for the metrics from Chapters 2–5. The choice is influenced by your existing stack (already on W&B for training? Weave is friction-free), your instrumentation preference (OpenInference/OTel standardization? Phoenix), and your product-team's UI needs (heavy annotation workflow? Langfuse's annotation queue is mature). Rearchitecting between them is a real cost; picking one and committing is usually the right move.

## The instrumentation contract

Regardless of platform, the eval discipline in this module needs specific fields on every LLM span for the downstream metrics to be joinable. The instrumentation contract:

- **`session.id`** — the session-level cluster identifier from Chapter 3's cluster-bootstrap. Every span in a session shares this id.
- **`user.id`** (or a hashed pseudonym in privacy-restricted environments) — the user-level identifier from Chapter 4's user-randomized A/B.
- **`model.name`** and **`model.version`** — the exact model version served for this call. Model routing that returns "the current default model" without version-pinning breaks A/B and shadow analyses.
- **`experiment.id`** and **`experiment.arm`** — for calls that landed in an active A/B, the experiment identifier and which arm this call belonged to. Missing this field makes the A/B analysis in Chapter 4 impossible to reconstruct.
- **`traffic.slice`** — a coarse categorization of the traffic (product surface, geo, tier). Enables the shadow-stratification and A/B-segmentation from Chapters 3 and 4.
- **`prompt.template.id`** and **`prompt.template.version`** — the system-prompt template SHA. Prompt changes without version bumps are the second-most-common source of "why did the metric change?" mysteries (after judge drift).
- **`evaluator.name`** and **`evaluator.version`** and **`evaluator.backend.model.version`** — on every evaluation attached to a span, the evaluator identity is versioned. Judge drift monitoring (below) depends on this.

An instrumentation contract that omits any of these fields will hobble the eval workflows in the rest of this module. Enforce it as a code-review discipline; the OpenInference spec's optional-attributes model does not do this for you.

## Drift alerts: three kinds, three designs

"Drift" is a loaded word in this space because three different things are called drift, and they require different measurements.

### Input drift (data drift)

The distribution of *inputs* to the model has changed. Users have started asking about a topic they were not asking about last month; a new marketing campaign has shifted the query length distribution; a competitor product outage has flooded traffic into a specific product surface.

Input drift is *not* a model regression on its own — the model may be handling the new distribution fine. But it is a signal that other metrics (quality, safety, latency at percentiles) should be reviewed on the shifted distribution, because your baselines were established on the old distribution.

Measurement: population-level distribution shift statistics on request features. Common choices:

- **Population Stability Index (PSI)** on histogram-binned features (query length, hour-of-day, product surface, language). Alerts at PSI > 0.1 (moderate) or > 0.25 (major) are the standard thresholds; these are eyeballed heuristics rather than statistically-derived, but they are widely used.
- **Embedding drift** via distance between the centroid of the current window's request embeddings and a reference window's centroid, or via a two-sample kernel test (MMD — Gretton et al. 2012) on the embedding sets. Phoenix's embedding-drift view uses this shape.
- **Categorical distribution shift** via chi-squared or Jensen–Shannon divergence on top-N categorical features.

Alerting design: input drift alerts are usually **informational**, not paging. They land in a daily digest, they annotate the confidence-sequence dashboard from Chapter 5 (so an on-call diagnosing a quality alert can see that inputs shifted at the same time), and they trigger a rebaseline conversation rather than an incident.

### Output drift (model behavior drift)

The distribution of *outputs* from the model has changed even though the model is nominally the same version. Causes: silent vendor updates on hosted models (the underlying model behind a name-only route was replaced), a retrieval index reindex, a shift in the top-K distribution from a rerank change, a prompt template update that was not versioned.

Measurement: same tools as input drift but applied to the *outputs* — output length distribution, output embedding drift, refusal rate on production traffic, format-compliance rate. When output drift shows up without a corresponding input drift, the causal candidates are the model, the prompt, the retrieval, or the tool responses — a specific list to walk.

Alerting design: output drift alerts are more urgent than input drift alerts because they often precede a quality regression by hours or days. Route to the eval channel with a P3 (investigate today), and if the drift correlates with a subsequent metric regression, promote to P2 or P1 per the runbook.

### Metric drift (quality / safety metric shift)

The metric itself has shifted — the sequential monitor from Chapter 5 has fired. This is the alerting shape that this module has been building toward.

Measurement: the confidence sequence on the metric, per the previous chapter.

Alerting design: the sequential monitor's α budget is set by severity (safety = 0.001, quality = 0.01). The runbook fires from the alert, and the diagnostic is *always* to look at the input-drift and output-drift signals alongside the metric alert. Three common resolutions:

- **Confirmed model regression.** The metric moved because the model got worse. Roll back to the incumbent (if the change was a ramp) or promote the incident to a launch review.
- **Traffic shift attribution.** The metric moved because the traffic composition shifted. The model is fine on the like-for-like segments; the aggregate moved because a bad segment now dominates. Fix: re-report the metric on stratified segments, and either the aggregate recovers or a specific segment's underperformance becomes the actionable finding.
- **Judge drift.** The metric moved because the evaluator changed. See the next section.

## Judge-drift alerting: the special case

Judge drift is called out as its own concept in Chapter 5, and it belongs in the observability chapter because the platform is what makes it detectable in the first place.

The failure mode: an LLM-as-judge grades your quality metrics. The judge is hosted (Claude, GPT, Gemini) or self-served (Llama-family). The judge's underlying model can change — the vendor rolls a point release, the routing changes, a self-served model is re-fine-tuned. When the judge shifts, every downstream quality metric shifts, and the shift is silently attributed to the target model.

Design: **a judge-canary dataset**, small and fixed, with human-gold labels, that the judge evaluates continuously alongside its production duty. Concretely:

- Assemble a canary set of 100–500 `(prompt, response)` pairs across the categories the judge grades (helpfulness, correctness, refusal-detection, format-compliance, whichever domain the judge covers). Each pair has a human-gold label reviewed by two annotators; disagreements adjudicated.
- Run the current-production judge across the canary set on a schedule (daily, hourly, or continuously if the judge is cheap).
- Compute the judge's agreement rate with the gold labels. A confidence sequence on this rate is the judge-drift monitor.
- Alerting threshold: absolute — alert if the upper bound of the agreement CI falls below the pre-committed threshold (e.g., 0.85 for a well-calibrated judge that historically runs at 0.90+).

When the judge-canary monitor fires:

- All downstream metrics graded by that judge are suspect until the judge is verified.
- The vendor's release notes are checked for a recent model update.
- If a vendor model update is confirmed, either (a) pin the judge to the pre-drift version (Anthropic's dated model IDs, OpenAI's snapshot IDs) if possible, or (b) accept the new judge behavior and re-baseline the downstream gates and thresholds as a documented event. Silent acceptance is the failure mode.

Judge drift is the most operationally-important observability signal in an LLM eval stack, and it is invisible without an explicit monitor. Every one of the three platforms above supports the shape — Phoenix's experiment framework, Langfuse's dataset-plus-evaluator pattern, Weave's `weave.evaluate` — but none of them will build the canary set for you.

## The dashboards a launch owner reads

The endpoint of the observability integration is a small number of dashboards that a launch owner and an on-call read on a regular cadence:

- **The always-on quality-and-safety dashboard.** One line per confidence-sequence monitor (per metric per slice per model version), with the current value, the CI, the threshold, and the last-alert timestamp. This is the pager view.
- **The candidate-vs-incumbent comparison view.** For an active A/B or shadow, the primary metric plus guardrails, split by segment, updated live. This is the launch-review view.
- **The judge-canary status page.** One line per judge in production use, with the agreement rate on the canary set, the CI, and the last vendor-model-version pin. This is the eval-team's daily-standup view.
- **The trace explorer.** Not a summary — a queryable view of individual traces filtered by user/session/experiment/slice, with evaluations inline. This is the root-cause view when an alert fires.
- **The dataset-and-eval library.** A registry of the datasets used by the offline regression suite, the shadow, and the A/B evaluators, with revision history and last-run status.

A team that has these five views in place, all pointing at a single traces-plus-evaluations backend, has closed the observability loop. A team that has any of them missing — or worse, has three of them in three different tools that do not share identifiers — will regularly find that the alert cannot be root-caused because the trace has aged out, or the metric moved because the judge was upgraded and nobody noticed.

## The alert-fatigue trap and how to avoid it

Every observability chapter must at some point say: **an alert that pages the on-call twice a week and requires no action either time is worse than no alert at all.** Alert fatigue is a real operational failure mode, and it is the most common way an observability integration destroys its own value.

Practical rules:

- **Every alert has a runbook.** The runbook says "when this alert fires, do exactly these steps; the expected outcome of the diagnostic is either X, Y, or Z; each of those leads to a specific next action." Without a runbook, the on-call cannot act, and the alert becomes noise.
- **Every alert has a severity that matches the α budget.** A monitor at α = 0.001 with an infrequent-alert budget is a page; a monitor at α = 0.05 exploring drift is a daily-digest item. Do not page on α = 0.05.
- **Every alert is periodically re-tuned.** A monthly review of alert firings: which alerts fired, how many were true positives, how many led to an action. Alerts with a low true-positive rate get their thresholds tightened or their α budgets shrunk; alerts that never fire get their thresholds loosened or are removed.
- **False-positive audits are logged.** When an alert fires and turns out to be a false alarm, the reason is recorded (traffic shift, one-off cascading upstream incident, transient network blip). Patterns in the false-alarm log are where alert design improves.

Chapter 5's α discipline is the statistical version of this; the operational version is the runbook and the alert-audit.

## Guidance for the eval author

- **Pick one platform and commit.** Multi-platform eval infrastructure is a duplication burden and a source of subtle metric divergences; the platform is the join point.
- **Enforce the instrumentation contract.** Session id, user id, model version, prompt version, experiment arm, evaluator version — every LLM span carries these or the downstream analysis is degraded.
- **Judge-canary monitor from day one.** The single most valuable always-on monitor in an LLM eval stack; without it, judge-drift metric changes are attributed to the target model.
- **Three drift categories, three designs.** Input drift informational, output drift investigative, metric drift paging with severity by α.
- **Runbooks for every alert.** No runbook, no alert.
- **Retention long enough to root-cause.** Trace retention must be longer than the longest realistic time-to-diagnose (usually 30 days minimum, longer for regulated environments).
- **Version everything.** Prompts, evaluators, datasets, model IDs. The traceability requirement is what makes the observability stack an eval instrument rather than a log store.

## Summary

Production observability platforms — Arize Phoenix, Langfuse, W&B Weave, and their peers — provide the traces-and-evaluations substrate that the four altitudes of this module have been producing data for. All share the same primitives (traces, spans, evaluations, datasets) but differ in ecosystem fit and specific feature emphasis. The eval discipline requires a specific instrumentation contract on every LLM span (session, user, model version, prompt version, experiment arm, evaluator version) without which the downstream shadow, A/B, and monitor analyses lose their traceability. Drift comes in three kinds — input, output, and metric — with three alerting designs at three severity levels. The single most valuable always-on monitor is a judge-canary that catches vendor-side judge drift before it silently shifts every downstream quality metric. The alert-fatigue trap kills observability integrations more often than technical failures do; every alert requires a runbook, a severity that matches its α budget, and a monthly re-tuning discipline. The next and final chapter of this module takes a step to the side of quality and safety metrics and covers the serving-time performance axis: MLPerf Inference's methodology for measuring TTFT, TPOT, and throughput under an accuracy floor.
