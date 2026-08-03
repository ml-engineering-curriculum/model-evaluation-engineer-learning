# Choosing And Routing Judges: Hosted, Open-Source, And Trained

Every choice this module has covered — rubric design, bias controls, calibration, arena aggregation — has assumed some judge model at the end of the pipeline. This chapter is about that choice: hosted frontier APIs (GPT-4-class, Claude-Opus-class), open-source general-purpose judges you run yourself, and dedicated *trained judges* like Prometheus and JudgeLM that are fine-tuned specifically for rubric grading. Each tier has a distinct cost profile and a distinct quality profile, and by 2026 the routing decision — which judge sees which item — is a real production concern, not a research curiosity.

## The three tiers

**Tier 1: hosted frontier judges.** Frontier chat models from major API providers, prompted with the rubric. This is the default in the LLM-as-judge literature (Zheng et al. 2023's original MT-Bench work used GPT-4 as the judge) and the default in most production stacks. Characteristics:
- Highest judge-vs-human agreement on most rubrics out of the box.
- Highest cost per call (order $0.001–$0.020 per judgment at 2026 pricing, depending on prompt length).
- Model version can silently change under you — a "gpt-4o" endpoint is not a fixed artifact — which means every judge-driven metric needs a recalibration cadence tied to provider version bumps.
- No control over the weights, so you cannot fix a judge-specific failure mode; you can only prompt around it.

**Tier 2: open-source general-purpose judges.** Open-weights instruction-tuned models (Llama-3-Instruct, Qwen-2.5-Instruct, Mistral-Instruct, DeepSeek-V3-class) run on your own infrastructure, prompted with the rubric. Characteristics:
- Judge-vs-human agreement typically 5–15 κ-points below the frontier at similar prompt lengths, task-dependent.
- Cost dominated by inference infrastructure, not per-call fees. At scale this can be an order of magnitude cheaper than frontier APIs.
- Weights are pinned; the judge does not silently change unless *you* change it.
- Full control over decoding, prompt handling, and quantization trade-offs.

**Tier 3: dedicated trained judges.** Models fine-tuned specifically to score rubric responses. Prometheus (Kim et al. 2024, Prometheus 1 and Prometheus 2) is the reference; JudgeLM (Zhu et al. 2023) and PandaLM (Wang et al. 2024) are earlier siblings; Auto-J (Li et al. 2024) is in the same family. Characteristics:
- Judge-vs-human agreement on rubric-scoring tasks approaches frontier levels at a fraction of the compute — Prometheus 2 reports parity with GPT-4 on several rubric benchmarks at 7B and 8x7B scales.
- Ships with a specific rubric prompt format (Prometheus expects a rubric with numeric anchors and reference); using it outside that format weakens the trained-in inductive bias.
- Weights are published; you run them yourself with the same infrastructure trade-offs as tier 2.
- Weakest tier when the rubric shape is far from what the model was trained on; strongest tier for rubric-shaped tasks it was designed for.

The right question is not "which tier is best" but "which tier's cost-quality point matches this item's stakes and difficulty."

## When each tier wins

The rough decision surface, keyed off two axes — how consequential the judgment is, and how ambiguous the item is:

- **High-stakes + high-ambiguity item** (release-gating eval on a novel rubric): tier 1 (hosted frontier), often with a panel of two judges from different families. Cost is dominated by the decision cost, not the token cost.
- **High-stakes + low-ambiguity item** (safety label, format-compliance): tier 3 (dedicated judge) or tier 2 (OSS general) is often sufficient after a calibration study confirms it, at 10–100× cost savings.
- **Low-stakes + high-ambiguity item** (regression alerting on a subjective quality rubric): tier 1 is overkill; tier 2 or 3 with a monitored κ against a small human validation set is the pragmatic choice.
- **Low-stakes + low-ambiguity item** (bulk grading on a well-defined rubric with clear labels): tier 3 or tier 2, cheapest defensible choice.

None of this replaces Chapter 5's calibration study. The tier is a *hypothesis* about where the cost-quality point is; the calibration study is the evidence that supports (or rejects) the choice for your specific task.

## The routing decision

Once you have more than one judge tier available, the routing question is: given a new item to grade, which judge do you send it to? Three routing patterns that recur across production stacks.

**Fixed-tier routing.** The simplest: send everything to a single tier. Fine when the eval is small enough that cost is not the bottleneck, or when the eval set is homogeneous enough that per-item variability does not justify tier switching. This is the default for most internal evals and should be the default until you have a reason to complicate it.

**Difficulty-based routing.** Send easy items (unambiguous, high judge confidence) to a cheap tier; escalate hard items (ambiguous, low judge confidence, or judge disagreement across orders in the swap-and-average check) to an expensive tier. Two common escalation triggers:
- Swap inconsistency: if the cheap judge returns a different verdict on `(A, B)` and `(B, A)`, escalate.
- Low judge-reported confidence: some judges will emit a confidence score alongside the verdict (either directly, or by taking the log-probability of the verdict token). Below a threshold, escalate.

The cost model is: `total_cost = cheap_rate × cheap_cost + (1 - cheap_rate) × expensive_cost`, and the interesting quantity is the *escalation rate* — the fraction of items that end up on the expensive tier. A well-tuned routing system might resolve 70–80% of items on the cheap judge and escalate the remaining 20–30%, capturing most of the cost savings while preserving frontier-tier accuracy on the hard items.

**Panel routing.** Send every item to two or more judges (from different families, ideally); use them as a mini-arena and take the majority. This is not really "routing" — it is redundancy — and it is the most defensible pattern for high-stakes evals where a single judge's family bias could contaminate the result. The cost multiplier is the number of judges in the panel; the quality lift is real and documented (Panickssery et al. 2024, and internally reported by many teams: panel-of-two typically beats single-judge κ against humans by 5–10 points).

Compose these three: use fixed-tier routing for a large low-stakes eval; use difficulty-based routing to bring average cost down on the medium-stakes eval; use panel routing on the small high-stakes eval that gates a release.

## Cost model: what the numbers look like

A worked example so the tier trade-off is concrete. Suppose an eval set of 10,000 items, a rubric that produces ~1,000 input tokens and ~200 output tokens per judgment, and the following per-tier costs (indicative, 2026):

- Tier 1 (frontier hosted): ~$0.005 per judgment. 10,000 items = **~$50**.
- Tier 2 (OSS general-purpose, self-hosted): amortized ~$0.0005 per judgment at moderate scale. 10,000 items = **~$5**, plus fixed infrastructure cost.
- Tier 3 (dedicated trained judge, self-hosted, ~8B parameters): amortized ~$0.0002 per judgment. 10,000 items = **~$2**, plus fixed infrastructure cost.

The per-run cost gap is 10× to 25×. On a single 10K-item eval this looks small in absolute dollars, and tier 1 is easy to justify. On a *daily* eval — regression testing over 10K items every day for a year — the tier gap is $18K vs $700, and the routing question matters.

Now add difficulty-based escalation: if 75% of items resolve on tier 3 and 25% escalate to tier 1, the per-item cost is `0.75 × 0.0002 + 0.25 × 0.005 = 0.00140`. On 10K items × 365 days that is ~$5.1K instead of $18.3K, an ~65% saving. The escalation rate is the knob; below about 20% you approach tier-3-only cost, above about 60% you lose the routing benefit and should just run tier 1.

A cost model that only accounts for token spend misses two things:

- **Infrastructure fixed costs.** Self-hosted tiers 2 and 3 require a GPU allocation whether you use it or not. At small volumes hosted tier 1 can be *cheaper* than self-hosted, because the hosted provider amortizes the GPU across many tenants.
- **Human review cost on escalated / low-confidence items.** For high-stakes decisions, the tail of items the judge is uncertain about often goes to a human. Human review is $2–$10 per item, which dominates any tier-1-vs-tier-3 gap. The cost model should include a human-review budget for the tail.

A defensible cost model reports total cost per eval run under three scenarios (all-tier-1, all-tier-3, tuned-routing), the escalation rate under the routing scenario, and the κ against humans under each — because a cost saving that comes with a κ collapse is not a saving.

## Judge drift and version pinning

Frontier hosted judges change silently. A `gpt-4o` endpoint in March is not the same model as a `gpt-4o` endpoint in September; the release notes might not enumerate the changes; the calibration study that showed κ = 0.72 six months ago is no longer valid evidence about the judge's current behavior. This is not a hypothetical — it is the reproducibility incident that motivates version pinning in every serious eval stack.

Pragmatic disciplines:

- **Pin the specific model version if the provider offers one** (`gpt-4-0613` rather than `gpt-4`, `claude-3-5-sonnet-20241022` rather than `claude-3-5-sonnet`). Not all providers expose stable versions; when they do, use them.
- **Rerun the calibration study on a fixed anchor set every 30–90 days**, regardless of whether the provider announced a change. A shifted κ is your only reliable signal that something moved.
- **Prefer self-hosted judges for a benchmark whose numbers must be reproducible across years.** Tier 2 or tier 3 on pinned weights gives you a reproducible artifact; a hosted endpoint does not.
- **Publish the judge version and the calibration κ alongside every judge-scored number.** Numbers without judge provenance are unfalsifiable.

Self-hosted judges do not drift silently — you own the weights — but they can still change: a quantization tweak, a decoding-config change, an inference-server upgrade. Version-pin the *judge configuration*, not just the model weights.

## Panel of judges as a specific pattern

Panel-of-judges is worth naming because it is the most common high-stakes pattern and because it composes with all of the routing patterns above. The mechanics:

- Run the same rubric on the same item with 2–3 judges from different families (e.g., one hosted frontier + one OSS general-purpose + one trained judge).
- Aggregate by majority vote (odd panel size), or by unanimity (unanimity = ship; disagreement = escalate to human).
- Compute per-judge κ against humans on a calibration slice, and use the highest-agreement judge as the tie-breaker in a majority vote if you want to weight the panel.

Panel judging eliminates single-judge family bias by construction and reduces self-preference bias to near-zero when subject and judge families are disjoint. The cost is the panel-size multiplier and the aggregation complexity. Panel judging is generally not worth it for low-stakes evals; it is generally required for evals that gate a launch or a compliance decision.

## What to actually pick

For most teams standing up model-graded evaluation for the first time, a defensible starting stack:

- **First calibration run:** hosted frontier (tier 1). Understand the rubric-and-judge behavior with the strongest judge you have access to. Establish a κ ceiling.
- **First production eval:** hosted frontier at low volume, or OSS general (tier 2) if volume matters. Publish κ against humans on your calibration slice with the deployed judge, not just the calibration-run judge.
- **When cost matters:** add a dedicated trained judge (tier 3) — Prometheus is the current reference — as the *primary* judge on rubric-shaped tasks it was designed for. Route to tier 1 as the escalation on high-uncertainty items. Recalibrate on a monthly anchor set.
- **When stakes matter:** move to a panel of two judges from different families, majority vote, escalate disagreements to a human queue.

Do not adopt tier 2 or tier 3 without a calibration study. Do not stay on tier 1 out of inertia if a calibration study shows tier 2 or tier 3 reaches the same κ. Do not skip the recalibration cadence because "the judge was fine six months ago."

## Summary

Judge selection is a cost-quality decision across three tiers: hosted frontier APIs (highest baseline agreement, silent version drift, per-call cost), open-source general-purpose judges (self-hosted, weights pinned, lower baseline agreement), and dedicated trained judges like Prometheus and JudgeLM (frontier-competitive on rubric-shaped tasks, cheapest per call, most fragile off-distribution). Routing across tiers is a real production concern once eval volume is large: fixed-tier for simplicity, difficulty-based escalation for cost savings, panel-of-judges for high-stakes decisions. The cost model must account for infrastructure fixed costs and human review of the tail, not just token spend. Hosted judges drift silently and require a recalibration cadence tied to a fixed anchor set; self-hosted judges do not drift, but their configuration must be version-pinned. A judge score without a companion κ and a companion judge-version tag is not a defensible number. This closes the module: designing the rubric, controlling the biases, calibrating to humans, aggregating pairwise into system ratings, and choosing which judge does the work.
