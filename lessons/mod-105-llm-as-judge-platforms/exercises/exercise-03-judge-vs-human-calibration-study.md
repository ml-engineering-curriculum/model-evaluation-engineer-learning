# exercise-03: Judge Vs Human Calibration Study

**Estimated effort:** 4 hours

## Objective

Run a full judge-vs-human calibration study on your rubric from exercise-01 (or any equivalent rubric you use). Produce human labels on a 100+ item validation set with at least two annotators and an adjudication protocol, compute the appropriate agreement statistics (Cohen's κ for categorical, weighted κ for ordinal, or Spearman / Kendall for continuous), and make an explicit ship / rework / caveat decision for the judge against a stated threshold. The deliverable is a calibration report that a reviewer could use to trust — or refuse to trust — the judge's aggregate score.

## Prerequisites

- mod-105 Chapters 2, 3, 5 (rubric design, calibration).
- mod-102 Chapter 4 (inter-annotator agreement, adjudication).
- exercise-01 completed (you have a rubric artifact) OR a published rubric of comparable specificity.
- A judge model (any tier — this exercise measures whichever judge you have; the choice itself is exercise-05's problem).
- At least 2 human annotators. You can be one of them. The other can be a teammate, a mentor, or a friend willing to spend ~90 minutes labelling items on your rubric. Real production evals typically use paid annotators; for the exercise, informal recruitment is fine as long as the rubric is written down and the annotators are following it, not vibes.

## The validation set

Assemble `data/calibration.jsonl` — a validation subset of your eval covering 100–200 items. The set should be:

- **Representative of the deployed distribution.** Sample from real production data if you have it (with any PII scrubbed). If not, sample uniformly across the categories in your task.
- **Balanced enough for κ.** If your rubric's label distribution in the wild is 90/10 skewed, sample the minority label at a higher rate for the calibration set so κ is well-defined. You can weight-reverse in analysis.
- **Held out from any set the rubric was tuned against.** If you developed the rubric anchors against a pilot set (exercise-01), those items do not belong here.

Each item carries: `id`, `question` (or the equivalent input), `reference` (if applicable), and the model response to be judged.

## Requirements

### Part A — human labels

Ship `data/human_labels.csv` with columns: `id`, `annotator_a_label`, `annotator_a_notes`, `annotator_b_label`, `annotator_b_notes`, `adjudicated_label`, `adjudication_reason`.

Rules:

1. **Human labels first, judge last.** Neither annotator sees judge output before labelling. This is not negotiable.
2. **Rubric is shared, in writing.** Every annotator has the rubric prompt in front of them. The rubric they use is *exactly* the criteria and anchor language the judge sees.
3. **Independent labels.** Annotators do not confer on individual items during labelling. If they need to clarify the rubric, that is a rubric edit that gets version-bumped and applied uniformly.
4. **Adjudication protocol.** Written before labelling: if annotators disagree, adjudicate by one of (a) a third rater, (b) discussion between the two annotators, (c) a domain-expert rater. Document which. On low-stakes exercises option (b) is acceptable; on the report you write down explicitly that this is what you did.
5. **Time each annotator.** Record total time in the report — an annotator that took 5 seconds per item is not doing the rubric.

Compute and report **inter-annotator agreement** (annotator A vs. annotator B, before adjudication) using the same agreement statistic you will use for judge-vs-human. This is the *ceiling* on judge-vs-human agreement and the number you will compare judge-κ against.

### Part B — judge labels

Run the judge over the same 100–200 items using the same rubric configuration you would deploy. Include the bias controls from exercise-02 where appropriate — for pairwise rubrics, swap-and-average is required; for absolute rubrics, log-and-report length correlation. Save raw judge output + parsed labels as `logs/judge.jsonl`.

Do not rerun the judge to "improve" agreement after seeing the human labels. That is p-hacking and it invalidates the study.

### Part C — the agreement analysis

Pick the correct statistic for your rubric's output shape (Chapter 5):

- **Binary or unordered categorical:** Cohen's κ. `sklearn.metrics.cohen_kappa_score`.
- **Ordinal (e.g., 1–4 anchored):** weighted κ with quadratic weights. `sklearn.metrics.cohen_kappa_score(..., weights="quadratic")`.
- **Continuous or graded scores:** Spearman's ρ and Kendall's τ-b. `scipy.stats.spearmanr` / `scipy.stats.kendalltau`.
- **Pairwise verdicts (A/B/tie):** accuracy of judge-verdict vs. human-verdict, plus κ if the tie rate is meaningful.

Compute:

1. **Judge vs. adjudicated human label** — the primary agreement statistic.
2. **Judge vs. each annotator individually** — to check that the judge is not agreeing "on average" with humans by taking a middle path neither annotator would.
3. **Human vs. human (inter-annotator)** — the ceiling.
4. **Bootstrap 95% CI on every reported agreement number.** 1,000–10,000 resamples. Any agreement number reported without a CI is not accepted.
5. **Confusion matrix** — full judge-label × human-label matrix. This is the diagnostic; if the judge is systematically one anchor off, it will show up on the sub-diagonal.

Also report:

- Overall accuracy (fraction of items where judge matches adjudicated human).
- Per-slice κ / accuracy if you have obvious slices (topic, length bucket, difficulty).
- Any parse-failure rate — items where the judge output could not be parsed and were excluded.

### Part D — the ship decision

Structure your ship decision as the five-check list from Chapter 5:

1. **Is judge-human κ above the ship threshold for this metric's stakes?** State the threshold you chose (κ ≥ 0.5 for regression alerting, κ ≥ 0.7 for launch gating, κ ≥ 0.8 for compliance) and whether the measured κ meets it.
2. **Is judge-human κ within 0.1 of human-human κ?** If humans agree at κ = 0.55 and the judge is at κ = 0.50, the judge is measuring the ceiling. If humans agree at κ = 0.80 and the judge is at κ = 0.55, the judge is the limitation and rubric / judge rework is worthwhile.
3. **Are the systematic errors documented?** Read the confusion matrix. Name the disagreement modes: "the judge systematically over-scores responses with citations" is a testable claim. "The judge is worse on longer responses" is a testable claim. Vague claims are not acceptable.
4. **Are the biases from Chapter 4 controlled and reported alongside?** A κ that came out of an uncontrolled judge is optimistic.
5. **Is there a re-measurement plan?** Cadence, anchor set, who owns it.

The decision is one of three:

- **Ship as-is** — all five green.
- **Ship with caveats** — usable for a specific decision the report scopes, not for others.
- **Rework** — rubric or judge changes required before another calibration study.

### Part E — the report

Write `REPORT.md` (2–3 pages) covering:

1. **Task and rubric summary.** What is being scored, one paragraph.
2. **Study design.** Validation-set size and sampling, annotator count and background, adjudication rule, judge configuration and version.
3. **Inter-annotator agreement.** Number with CI. This is the ceiling.
4. **Judge-vs-human agreement.** Number with CI, plus the confusion matrix.
5. **Per-annotator judge agreement.** Two numbers, sanity check on Part 3.
6. **Bias-control state.** Which of Chapter 4's controls were applied, what residual biases showed up in the calibration set.
7. **Disagreement modes.** 3–5 items where the judge and humans disagreed most, with a short prose analysis of *why* for each. This is the section that turns a κ number into actionable rubric edits.
8. **Ship decision.** The five-check outcome and the resulting call.
9. **Re-measurement plan.** Anchor-set size, cadence, owner, trigger conditions (new judge model version, new task domain, etc.).

### Part F — bundle

Ship:

- `data/calibration.jsonl`
- `data/human_labels.csv` (with per-annotator + adjudicated columns; **redact any sensitive content** — if the items or labels are sensitive, publish only the aggregate agreement numbers and note the omission)
- `logs/judge.jsonl`
- `analysis/agreement.py` — reproducible computation of all reported numbers from the two files above
- `analysis/agreement_results.md` — tables and numbers
- `REPORT.md`
- `run.sh` — one-shot rerun of judge labels + agreement computation

## Starter guidance

- **A validation set smaller than 50 items is not a calibration study.** The CI on κ is too wide to distinguish "judge is trustworthy" from "judge is unreliable." If time-constrained, prefer 50 items with two annotators over 150 items with one annotator.
- **Human labels first, no exceptions.** If either annotator ever sees the judge's answer before recording their own, the study is contaminated and the κ is overstated.
- **The adjudication reason is worth writing down.** For items the two annotators disagreed on, one sentence on why. Future you (and the next person owning this rubric) will find these invaluable when the rubric needs a v2.
- **Report human-human κ prominently.** Half the calibration reports in the wild forget to include it, which turns judge-κ into an unanchored number. If humans agree at 0.6 and the judge is at 0.55, the judge is near-ceiling. If humans agree at 0.9 and the judge is at 0.55, the judge is the problem. You cannot tell without the human-human number.
- **Look at the confusion matrix even if κ is high.** A judge at κ = 0.75 that systematically off-by-ones on the middle two anchors of a 4-point scale is still shippable, but the diagnostic tells you what to fix in v2. A judge at κ = 0.75 that occasionally labels "correct" as "wrong" on high-severity items is not shippable regardless of the aggregate.
- **Bootstrap CIs are non-negotiable.** A judge at κ = 0.72 (0.65–0.79) is a shippable metric; a judge at κ = 0.72 (0.55–0.85) is not, even though the point estimate is the same. The bootstrap is the difference.
- **On skewed label distributions, report Krippendorff's α as a cross-check.** κ is prevalence-sensitive; α is less so. If your labels are 85/15, the two statistics can disagree, and the disagreement itself is informative.
- **Parse-failure items are not "just a data issue."** If 5% of judge outputs fail to parse, the judge cannot reliably follow the rubric format on your task. Fix the rubric prompt or the parser and rerun; do not silently drop them and quote the remaining κ.
- **You are not tuning the judge in this exercise.** If the calibration fails, the report should say so and enumerate what would change in the rework. The paired solutions repo may include a full re-tune cycle; the exercise is scoped to the *measurement* discipline.

## Acceptance criteria

- The validation set has at least 100 items (50 minimum for a genuinely constrained exercise, with the constraint noted) and is representative of the deployed distribution.
- Human labels are collected from at least two independent annotators, with an adjudication rule pre-registered and executed. `human_labels.csv` shows per-annotator and adjudicated columns.
- The correct agreement statistic for the rubric's output shape is used (κ / weighted κ / Spearman / Kendall / accuracy for pairwise) and reported with a bootstrap 95% CI.
- Inter-annotator agreement is reported alongside judge-human agreement.
- A confusion matrix is included and the report names at least two systematic disagreement modes in prose.
- The ship decision is explicit and structured against the five-check list — not a single-number verdict.
- The re-measurement plan names an anchor-set size, a cadence, and an owner.
- The bundle is reproducible: a reviewer with judge credentials can rerun the pipeline via `run.sh` and reproduce the agreement numbers within stochastic tolerance.

## Stretch goals

- **Per-slice calibration.** Compute κ per topic, per length bucket, and per any other slice with ≥ 20 items. Report the max-min per-slice κ delta; a judge that is at κ = 0.7 overall but κ = 0.4 on the "safety-critical" slice is not shippable for safety decisions even if the aggregate looks fine.
- **Rubric-vs-judge attribution.** Re-run the study with a strictly *stronger* judge (frontier tier) if your primary judge was mid-tier. If κ jumps by > 0.15, the judge tier is the bottleneck (queue this for exercise-05). If κ barely moves, the *rubric* is the bottleneck and you need a v2 rubric before more judge shopping.
- **Compare weighted κ to unweighted κ.** On an ordinal rubric, both are worth reporting. The gap between them tells you whether the judge's misclassifications are mostly one-step (weighted κ close to unweighted, judge is nearly-there) or several-step (weighted κ much higher than unweighted, judge is missing badly and unweighted κ is punishing correctly).
- **Krippendorff's α across all three raters (annotator A, annotator B, judge).** Treats the judge as a "third annotator" and asks how well the three-rater panel agrees. Useful when the ship question is "can the judge replace a human in a three-rater panel."
- **Longitudinal anchor set.** Freeze a 30-item anchor subset. Rerun the calibration study monthly and plot judge-κ over time; the first month a hosted judge silently changes, this plot moves. Sketch this as a Grafana / Metabase panel spec and describe the alert threshold.
