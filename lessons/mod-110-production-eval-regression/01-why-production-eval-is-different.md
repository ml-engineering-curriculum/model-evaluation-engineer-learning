# Why Production Evaluation Is Different

Every module before this one measured a model against a *frozen* eval set — a benchmark, a held-out slice, a gold-labeled batch, a red-team suite. The unit of work was "run the model over these `N` items, aggregate, report a number with a confidence interval." That discipline is load-bearing and everything in this module builds on it. But it is not sufficient for the question a launch review actually asks: *is this model safe to ship into production traffic, and once shipped, how will we know when it stops being safe?* Production eval is what closes that gap. It takes the offline artifacts from mod-101 through mod-109 and wires them into the release process (pre-launch gates, shadow comparisons, controlled experiments) and into the live serving path (continuous monitoring, drift alerts, serving-time SLO measurement).

This chapter frames the module. It names the four altitudes at which production eval operates, draws the line between capability measurement (everything upstream) and *decision-support* measurement (this module), and sets up the vocabulary that Chapters 2 through 7 use.

## The four altitudes of production eval

A release-quality production eval program is not a single system. It is four coordinated systems, each answering a different question, and each with its own failure mode when confused for one of the others:

- **Offline regression suite with SLO-mapped gates.** Runs on every candidate model, deterministically, in CI. Answers "does this model clear the pre-launch pass/fail bar on the metrics we have committed to leadership?" The gates are boolean (pass or fail), the sample size is fixed, and the datasets are pinned. Chapter 2 builds this.
- **Shadow / dark-launch comparison.** The candidate model is served alongside the incumbent on real production traffic without its output being shown to the user. Answers "on the actual request distribution we serve, how does the candidate compare to the incumbent on the metrics we can compute without user feedback?" The traffic is non-IID, the distribution shifts by hour and day, and the comparison must be paired at the request level to be defensible. Chapter 3 builds this.
- **Controlled A/B experiment.** A fraction of live traffic is routed to the candidate, and outcome metrics (both proxy metrics from evaluators and business metrics from downstream logging) are compared between arms with pre-registered hypotheses, variance reduction, and a decision rule. Chapter 4 builds this using CUPED (Deng, Xu, Kohavi, Walker 2013) as the reference variance-reduction technique.
- **Continuous monitoring with sequential inference.** Once the model is fully ramped, the eval does not stop — it moves to always-on drift, safety, and quality monitors that must be able to raise alerts *early* without inflating false-positive rate across the many looks they perform. Sequential testing and confidence sequences (Howard et al. 2021; Waudby-Smith & Ramdas 2020) are the tool for this. Chapter 5 builds this.

Two supporting systems sit alongside the four altitudes:

- **Observability and drift alerting** wire the four altitudes into a platform (Arize Phoenix, Langfuse, W&B Weave, or a self-hosted stack) so the metrics, traces, and judge outputs are queryable, alertable, and auditable. Chapter 6 walks the three reference platforms and the drift-alert designs that keep them useful. Importantly, this chapter also covers *judge drift* — when your LLM-as-judge grader silently changes behavior because the vendor rolled the underlying model.
- **Serving-time performance benchmarking** measures TTFT, TPOT, and throughput against an accuracy floor. This is not "capability eval," and it is not "A/B experiment." It is a distinct axis — the model's serving-cost characteristics — and it belongs in the launch review alongside the quality numbers. Chapter 7 walks MLPerf Inference's methodology and how to adapt it to your own serving stack.

A release decision reads all four altitudes plus the two supporting systems as a portfolio. A model can pass the offline regression gates, look fine in shadow, and still be a bad ship if the A/B shows a proxy-metric regression, or if the serving benchmark shows TTFT doubling at the P99. Chapter 8 does not exist; the assembly of all of these into a launch scorecard belongs in mod-112 (eval systems design).

## Capability measurement versus decision-support measurement

Every measurement in mod-101 through mod-109 was a capability measurement. It asked "how good is the model at task X?" and returned a number. The number was informative but rarely directly actionable — a 3-point ARC-AGI improvement does not automatically ship a model.

Production eval is a different genre: **decision-support measurement**. Every measurement in this module exists to answer a specific yes/no question tied to a specific action:

- Offline regression gates → "block the release, or promote it to the next stage?"
- Shadow comparisons → "is the candidate close enough to the incumbent to justify a controlled experiment?"
- A/B experiments → "do we ramp to 100%, hold, or roll back?"
- Continuous monitoring → "page the on-call, or let it ride?"
- Serving benchmarks → "does the new model fit our latency SLO at the throughput we serve?"

Because every measurement has an action attached, two disciplines from mod-101 that were merely good practice upstream become non-negotiable here:

- **Pre-registered decision rules.** Before you look at the number, you commit to what number will trigger what action. This is not academic ritual — it is what prevents "the number was 1.8% and our threshold was 2%, but the trend looks good and we shipped anyway" from becoming a systematic release-review pathology. Chapter 4 makes this concrete for the A/B setting; the same discipline applies at every altitude.
- **Calibrated false-positive control.** Every alert, every gate, every sequential test performs *multiple looks* at the data. Naive p-value thresholds inflate the false-positive rate to the point where the alerts are pure noise (the "always-on eval that pages the on-call daily" pathology). Chapter 5 introduces confidence sequences as the reference technique for taming this without waiting for a fixed sample-size batch.

Both disciplines are already familiar from mod-101 Chapters 3–4 (paired confidence intervals and Benjamini–Hochberg for slice reporting). This module escalates them from "good hygiene" to "structural requirement" because the cost of an under-controlled alerting system in production is real dollars, real pages, and real on-call attrition.

## Why offline-quality numbers do not transfer to production

Two properties of production traffic break the assumption that "a benchmark improvement is a product improvement":

**Property 1: production traffic is non-IID.** The benchmark you evaluated on is drawn once, from a distribution that is fixed at authoring time. Production traffic drifts by hour of day, by day of week, by product surface, by cohort, by whether a marketing campaign is live, by whether a competitor product had an outage that day, and by the demographic composition of who happened to be online. A model that is uniformly better on the benchmark may be worse on the segment of traffic that dominates during your peak hours. Chapter 3 walks through how a shadow comparison quantifies this — and why the naive "one number for the whole day" answer is misleading.

**Property 2: production users respond to the model.** In an offline eval, the user's next request does not depend on the model's previous response. In production, it does — every interaction changes the distribution of the next one, and a model that is subtly worse at the first turn will systematically be handed harder second turns, and worse third turns. This is the "interference" problem that classical A/B experiment design assumes away (SUTVA — the stable-unit-treatment-value assumption). It shows up specifically in LLM products with multi-turn sessions, in recommenders, and in any system where the treatment affects the distribution of the control's future requests. Chapter 4 discusses when SUTVA holds and when it does not, and what to do about it (session-level randomization, cluster randomization, longer holdouts).

Both properties are the reason the offline gate is *necessary but not sufficient*. The offline gate answers "did the model regress on the fixed things we care about?" The shadow, the A/B, and the continuous monitor answer "does it hold up when reality is doing what reality does?"

## Where safety metrics live in this stack

Safety and red-team metrics from mod-109 are *the highest-cost-of-silent-regression class of metric* in this module. A capability regression on TriviaQA ships as a slightly worse product; a refusal-rate regression on the self-harm category ships as a headline. That asymmetry has two consequences for production eval:

- **Safety metrics live in the offline regression suite as blocking gates, not warning gates.** A drop in refusal rate on any policy-violating category below the pre-committed threshold blocks the release. Chapter 2 walks the "pass/fail" versus "warn/watch" gate distinction.
- **Safety metrics live in the continuous monitor with tighter thresholds and lower alert-fatigue tolerance.** The continuous-monitoring discipline in Chapter 5 must set alpha budgets that reflect the severity of a miss on safety metrics, not just the sampling cost. The same false-positive rate that is fine on "helpfulness score" is not fine on "refusal rate on weapons prompts."

The safety-eval measurements themselves live in mod-109; how they are wired into the release process lives here.

## Where this module sits in the track

Prerequisites you should have solidified:

- **mod-101 (evaluation foundations).** Every gate, every comparison, every alert is a proportion or a difference of proportions with a confidence interval. The bootstrap CI, paired comparisons, and Benjamini–Hochberg slice reporting from mod-101 are the machinery this module wires into pipelines.
- **mod-105 (LLM-as-judge platforms).** The proxy metrics used in shadow and A/B are almost always judge-graded — helpfulness, correctness, tone, format-compliance. The judge bias controls from mod-105 apply here, plus one specific to production: *judge drift*, when the underlying judge model rolls under you. Chapter 6 makes this concrete.
- **mod-109 (safety and red-team eval).** Safety metrics have unique severity and unique reporting rules; the release-blocking discipline in this module assumes those measurements exist.

Downstream modules that this one feeds:

- **mod-111 (eval platform engineering).** The observability integrations in Chapter 6 depend on a platform's data model — spans, traces, evaluator wiring, retention policy. Mod-111 designs that platform end-to-end; this module treats the platform as a given and shows what the eval side needs from it.
- **mod-112 (eval systems design).** The end-to-end launch scorecard — how the four altitudes and the two supporting systems compose into a single go/no-go decision — is the systems-design capstone. This module supplies the components; mod-112 assembles the whole.

## What this module does not cover

Two adjacent disciplines are close enough that they need explicit disclaimers.

- **General A/B experimentation platform engineering** (variant assignment services, exposure logging, SRM checks at the platform level, feature-flag infrastructure). Chapter 4 assumes such a platform exists and focuses on the *eval-specific* choices: what metrics to compare, what variance-reduction to apply, what pre-registration to enforce. If you need to build the platform, Kohavi, Tang & Xu 2020 ("Trustworthy Online Controlled Experiments") is the reference and mod-111 covers the platform side.
- **Site reliability engineering / on-call practice.** Chapter 5 discusses alerting thresholds and Chapter 6 discusses observability platforms. Neither of those is a substitute for a real SRE practice — runbook engineering, alert routing, on-call rotation design, incident response. If your eval alerts are getting paged directly to a shared #alerts channel with no runbooks, the fix is process, not statistics.

## A note on tooling volatility

The observability space (Chapter 6) and the serving-benchmark space (Chapter 7) are the fastest-moving parts of this module. Arize Phoenix, Langfuse, and W&B Weave were the three widely-adopted open-source LLM observability platforms at the time of writing (late 2025 into 2026); MLPerf Inference is on v5.x and the LLM-serving submissions have converged on a small set of reference workloads (Llama-2-70B, Llama-3.1-405B, Mixtral, and updated GPT-scale references) with well-defined TTFT and TPOT quality-of-service constraints. Treat these as *versioned anchors*, not permanent fixtures. The discipline — a platform that gives you spans, traces, evaluator wiring, and drift alerts; a serving benchmark that separates the throughput axis from the latency axis with an accuracy floor — is durable. The specific vendor / benchmark version is not.

## Summary

Production evaluation is where the offline evaluation artifacts from mod-101 through mod-109 meet the launch process and the live serving path. It operates at four altitudes — offline regression with SLO-mapped gates, shadow / dark-launch comparison on production traffic, controlled A/B experiments with CUPED variance reduction, and continuous monitoring with sequential inference — plus two supporting systems (observability integration and serving-time benchmarking). Each altitude is decision-support measurement: every number is attached to an action, and each altitude has its own failure mode when confused for another. The two properties of production traffic that break naive "benchmark → product" reasoning are non-IID drift and interference between users; the discipline of pre-registered decision rules and calibrated false-positive control is how the module keeps releases honest in the face of both. Safety metrics from mod-109 sit at the highest severity tier and act as blocking gates and tighter monitors, not warning signals. The next chapter builds the first altitude: the offline regression suite whose gates map, one to one, to product SLOs and safety policy commitments.
