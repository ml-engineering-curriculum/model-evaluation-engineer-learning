# exercise-05: Judge Tier Routing And Cost Model

**Estimated effort:** 3 hours

## Objective

Build and defend a *tiered judge routing* pipeline for the rubric from exercise-01 (or an equivalent). Calibrate at least one hosted-frontier judge (tier 1), one OSS general-purpose judge (tier 2), and one dedicated trained judge (tier 3, e.g., Prometheus 2) against the same human gold set from exercise-03. Then implement difficulty-based escalation and produce a cost model that reports total cost, average-per-item cost, and κ-vs-humans under three routing scenarios: all-tier-1, all-tier-3, and tuned routing. The deliverable is a defensible tier-routing recommendation with a numeric trade-off table backing it.

## Prerequisites

- mod-105 Chapters 1, 5, 7 (judge modes, calibration, tier selection).
- exercise-01 (a rubric) and exercise-03 (a calibration set with human labels) are strongly recommended prerequisites; you will reuse the human labels from exercise-03 as the calibration ground truth here.
- Access to:
  - **Tier 1:** a hosted frontier judge (GPT-4-class, Claude-Opus-class, Gemini-Pro-class).
  - **Tier 2:** at least one OSS general-purpose instruction-tuned model you can run yourself (Llama-3-Instruct at ≥ 8B, Qwen-2.5-Instruct, Mistral-Instruct, etc.). Inference through vLLM, TGI, llama.cpp, or an OpenAI-compatible endpoint.
  - **Tier 3:** at least one dedicated trained judge model. [Prometheus 2](https://github.com/prometheus-eval/prometheus-eval) is the reference; JudgeLM, PandaLM, or Auto-J are alternatives. Follow the model's prescribed rubric format — do not adapt your rubric shape to the model's incompatible priors without noting the compromise.
- The bias-controlled harness from exercise-02.

## The dataset

Reuse `data/calibration.jsonl` and `data/human_labels.csv` from exercise-03. If exercise-03 was not completed, run a compressed version of it first: 50 items minimum, two annotators, adjudicated labels. The tier-routing decision *requires* human ground truth per item — every claim in this exercise is grounded against it.

Also assemble `data/production_shadow.jsonl` — a larger set of 500–2000 items *without* human labels, drawn from your production distribution, used only for the cost-projection scenarios. It must not overlap with the calibration set.

## Requirements

### Part A — per-tier calibration

For each of the three tiers (T1, T2, T3), run the judge over the calibration set with the appropriate rubric configuration:

- Apply the bias controls from exercise-02 (swap-and-average for pairwise; length-conditional reporting for absolute).
- For tier 3 (dedicated judge), use the model's *native* rubric format — Prometheus 2 expects `orig_instruction`, `orig_response`, `orig_reference_answer`, `orig_criteria`, and `orig_score_rubric` fields. Do not attempt to force it through a Prometheus-incompatible rubric shape; if your absolute rubric doesn't map cleanly, note the compromise and any *quality cost*.
- Log per-item raw output, parsed label, and per-call token cost (input + output).

Then, per tier, compute:

- **κ against adjudicated human labels** (weighted κ for ordinal rubrics; Cohen's κ for categorical; Spearman / Kendall for continuous). With bootstrap CI.
- **Accuracy** against humans as a supplement.
- **Confusion matrix** per tier.
- **Per-call cost** — for hosted, price × tokens; for self-hosted, compute an amortized cost from GPU hourly rate × time per call. Document the amortization model.
- **Per-call latency** — p50 and p95.
- **Parse-failure rate.**

Ship the per-tier results as `results/tier1.md`, `results/tier2.md`, `results/tier3.md`.

### Part B — difficulty signal

Design a *difficulty signal* the routing system uses to decide whether to escalate an item from a cheap tier to an expensive one. Two candidate signals from Chapter 7, or a combination:

- **Swap inconsistency.** If the cheap judge returns different verdicts on `(A, B)` and `(B, A)` in a pairwise rubric, escalate.
- **Low judge-reported confidence.** If the judge emits a confidence score (some rubrics ask for one) or if the log-probability of the verdict token is below a threshold, escalate.
- **Label-space margin.** For absolute rubrics with numeric scores, distance from the anchor description in the reasoning text, or a heuristic like "answer chose the middle anchor" as a signal for ambiguity.

Ship `routing/difficulty.py` implementing your chosen signal(s) and returning `(is_hard: bool, reason: str)` per item.

Validate the signal against ground truth: on the calibration set, items your signal flags as "hard" should have *lower* tier-3 κ than items it flags as "easy." If the signal is uncorrelated with per-item accuracy, it is not a useful difficulty signal and you need a better one.

### Part C — the routing scenarios

Implement three routing scenarios and score each end-to-end on the calibration set:

1. **All-tier-1 (baseline high-quality).** Every item to tier 1.
2. **All-tier-3 (baseline low-cost).** Every item to tier 3.
3. **Tuned routing.** Route to tier 3 by default, escalate to tier 1 when the difficulty signal fires. If tier 2 fits in your setup, use it as an intermediate tier.

For each scenario, compute over the calibration set:

- **Aggregate κ against humans.**
- **Total judge cost** (sum of per-call costs).
- **Average per-item cost.**
- **Total judge latency** (sum, useful for batch budget planning).
- **Escalation rate** (only meaningful for tuned routing).

Then project the same three metrics to the production-shadow set (using the trained routing policy from the calibration study — do not re-fit the policy on the shadow set).

### Part D — the cost model

Ship `analysis/cost_model.py` producing a `results/cost_model.md` table:

| Scenario | κ vs humans (95% CI) | Escalation % | Total cost | Cost per item | p95 latency |
|----------|---------------------|--------------|-----------|---------------|-------------|
| All tier 1 | ... | 0 | ... | ... | ... |
| All tier 3 | ... | 0 | ... | ... | ... |
| Tuned routing | ... | X% | ... | ... | ... |

Include:

- **Fixed infrastructure cost note.** If you self-host tier 2 or tier 3, the GPU cost is fixed regardless of usage. State the amortization assumption (e.g., GPU rented for the study at $X/hour, ran for Y hours). At low volume the hosted tier can be cheaper than self-hosted.
- **Break-even analysis.** At what total item volume does self-hosted tier 3 become cheaper than tier 1? Plot cost vs. volume for each scenario.
- **Human review budget for the tail.** For a hypothetical "review any item where the tuned pipeline is uncertain," compute the additional cost of human review at $2–$10 per item on the escalated-but-still-low-confidence subset.

### Part E — judge-drift risk register

Chapter 7 warns that hosted-tier judges silently change under you. For each tier in your stack, document:

- Model version pinned (or noted as unpinned).
- Recalibration cadence (monthly / quarterly / per-version-bump / never).
- Owner responsible for the recalibration.
- Trigger conditions for an unscheduled recalibration (a κ drop > 0.05 on the anchor set, a provider release-note that mentions the model, a change in rubric wording).

Ship as `results/drift_register.md` — one row per tier.

### Part F — the report

Write `REPORT.md` (≤ 3 pages) covering:

1. **Task summary.** One paragraph, referencing exercise-01 and exercise-03 outputs.
2. **Per-tier calibration.** Table of κ / accuracy / cost / latency / parse-failure per tier.
3. **Difficulty signal.** What signal you used, its validation against ground truth (per-slice κ on "easy" vs. "hard" items).
4. **Routing scenarios.** The cost-model table plus one paragraph interpretation per scenario.
5. **Break-even analysis.** The volume plot and the point at which tuned routing beats all-tier-1.
6. **Recommendation.** The tier-routing configuration you would ship for this task, justified against the numbers. Not "tier 3 is cheaper" — "tuned routing with tier 3 default and tier 1 escalation on X% of items achieves κ = Y and is Z× cheaper than all-tier-1, meeting the ship threshold set in exercise-03."
7. **Drift register.** Included as a table in the report.
8. **What would move the recommendation.** Under what conditions (new task, higher-stakes decision, different volume) would you move to a panel of two frontier judges or drop to all-tier-3? This is the section that turns your recommendation from "the answer today" into "the decision framework."

### Part G — bundle

Ship:

- `data/production_shadow.jsonl` (reference to exercise-03's calibration set is enough for that half)
- `judge/tier1.py`, `judge/tier2.py`, `judge/tier3.py` — per-tier judge harnesses (with rubric-format adapters)
- `routing/difficulty.py`, `routing/router.py` — routing signal + the tuned routing policy
- `logs/*.jsonl` — per-tier per-scenario judge logs
- `analysis/cost_model.py`
- `results/tier1.md`, `results/tier2.md`, `results/tier3.md`
- `results/cost_model.md`, `results/drift_register.md`, `results/cost_vs_volume.png`
- `REPORT.md`
- `run.sh` — one-shot rerun of the three scenarios

## Starter guidance

- **Do not skip tier 2 or tier 3 because "they'll be worse."** A calibration study that shows tier 3 reaches within 0.05 κ of tier 1 at 20× lower cost is one of the most valuable results you can produce. If you don't run the study, you cannot make that claim — and you'll default to tier 1 forever out of caution.
- **Prometheus (or your tier-3 judge) has a specific rubric format.** Read its documentation. Trying to force a Prometheus judge through an arbitrary rubric shape typically costs 10–20 κ points of agreement; using the native format recovers most of it. This is a *rubric adapter* problem, not a judge-quality problem.
- **The difficulty signal is the load-bearing part of tuned routing.** A signal that flags 60% of items as hard is not a routing signal — it's a random coin. A signal that flags 20% of items as hard and captures 80% of the disagreement is what makes routing work. Validate the signal empirically against ground truth per-item accuracy; do not ship a signal on intuition.
- **Fixed costs matter and are usually undercounted.** A GPU rented for one hour to grade 200 items is at least $2 amortized, likely more, and a small hosted API call at $0.005 apiece is $1 for the same 200 items. At small volumes hosted is often cheaper than self-hosted — this is a real result, not a research failure.
- **Escalation rate is a policy knob, not a natural constant.** You can move it by adjusting the difficulty threshold. The trade-off is cost (higher threshold = fewer escalations = cheaper = lower κ) vs. quality (lower threshold = more escalations = more expensive = higher κ). Plot the frontier and pick a point that meets the ship threshold at minimum cost.
- **Never claim a κ that came out of an uncontrolled judge.** All three tiers must have the bias controls from exercise-02 applied *before* their κ is compared. Comparing a bias-controlled tier 1 to an uncontrolled tier 3 inflates the tier 1 win.
- **Do not tune the routing on the calibration set and evaluate on the same set.** That is data leakage: your routing policy has already seen the ground truth. Tune on a subset, evaluate on the held-out subset, or use cross-validation. State the split.
- **The drift register is not optional.** Even for an exercise, writing down the recalibration cadence and the trigger conditions is the discipline that keeps a shipped judge honest six months later. Skipping this is how "the eval worked in March" turns into "the eval mysteriously broke in September."
- **A recommendation without a `what would move it` section is a decision, not a framework.** The point of the exercise is to build a defensible tier-routing decision that survives new information; the framework is what survives, not the specific tier choice today.

## Acceptance criteria

- Three judges from three tiers (T1 hosted frontier, T2 OSS general, T3 dedicated trained) are calibrated against the same human gold set from exercise-03, with per-tier κ (with bootstrap CI), accuracy, cost per call, latency, and parse-failure rate reported.
- The tier-3 judge is used in its *native* rubric format (or the compromise is explicitly noted and quantified).
- A difficulty signal is implemented and validated against ground truth (correlates with per-item disagreement).
- Three routing scenarios (all-tier-1, all-tier-3, tuned) are scored end-to-end on the calibration set, and the cost model projects them to the production-shadow set.
- The cost model includes fixed infrastructure amortization and a break-even volume plot, not just per-token costs.
- A drift register with recalibration cadence, owner, and trigger conditions is included for every tier in the stack.
- The `REPORT.md` closes with a specific ship recommendation grounded in the numeric trade-off table, plus a "what would move the recommendation" section.
- The bundle is reproducible via `run.sh`.

## Stretch goals

- **Add a panel-of-frontier scenario.** Run every item through two hosted frontier judges and majority-vote. Score it alongside the three main scenarios. Report the κ lift, the cost multiplier, and whether the panel resolves any of the previously-escalated hard items that tuned routing struggled with. Panel-of-two typically buys 5–10 κ points over single-judge; here you get to measure how much on your task.
- **Volume-tiered recommendation.** Extend the recommendation into a matrix: for < 100 items/day, do X; for 100–10,000/day, do Y; for > 10,000/day, do Z. This surfaces that the same task has different right answers at different scales.
- **Latency-aware routing.** Some items block a synchronous UI (regression alerting on a per-request basis); others are batch (nightly rollup). Extend the router with a latency budget per item: escalate to tier 1 only if latency budget permits, otherwise commit to tier 3. Report the trade-off in the cost model.
- **Rubric-adapter compatibility matrix.** For each of tier 1 / 2 / 3, document what happens to κ when you switch rubric formats (your custom rubric vs. Prometheus-native vs. MT-Bench-style). Publish the matrix. Rubric-format mismatch is one of the most common tier-3 disappointments and this makes it visible.
- **Cost-controlled online routing.** Simulate a live system with a monthly cost budget. Route items greedily to the cheapest tier that meets a per-item confidence threshold, escalating only when the budget-remaining / items-remaining ratio allows it. Report κ over the month; this is a realistic operational scenario the fixed scenarios miss.
- **Judge-drift injection experiment.** Simulate a hosted judge silently changing by swapping tier 1 to a different frontier model mid-experiment and rerunning the anchor calibration set. Measure the κ delta. This is the empirical case for the drift register.
