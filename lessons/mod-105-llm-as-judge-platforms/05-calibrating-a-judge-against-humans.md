# Calibrating A Judge Against Humans

A judge with a well-designed rubric and controlled biases is still, until proven otherwise, a plausible-sounding number generator. What turns it into a *measurement instrument* is calibration against humans: a study that measures the agreement between the judge's verdicts and a human gold set on your specific task, so you know how much to trust the judge's score before it gates a decision. This chapter is the mechanical part — which agreement statistic to use, how to size the study, how to interpret the result — plus the methodological part: deciding whether the judge is good enough to ship.

## The setup

You are calibrating one specific judge (rubric + model + version) against humans on one specific task. The artifact is:

- A *validation subset* of items from your eval set — typically 50–200 items, sampled to cover the task's distribution.
- *Human labels* on that subset, from at least two independent annotators, following the same rubric the judge uses.
- An *adjudication step* that resolves annotator disagreements into a single gold label per item.
- *Judge labels* on the same items, produced by the same rubric and configuration the judge will use in production.

The output is a set of agreement statistics between the judge and the adjudicated human labels, plus a decision — ship / caveats / rework — grounded in those numbers.

## Human labels come first

The order is not negotiable. Humans label the items *before* seeing judge scores. If humans see the judge's verdict first, their labels drift toward the judge's — this is the standard anchoring effect in annotation, and it silently inflates the agreement number.

Two humans, each labelling independently, following a written rubric. Krippendorff and IAA discipline from mod-102 Chapter 4 applies verbatim: shared rubric doc, held-out items, pre-registered adjudication rule for the disagreements. Budget 30–60 seconds per item per rater for a typical multi-sentence rubric; 100 items × 2 raters is roughly two hours of human time and is the minimum defensible study size for most tasks.

If your rubric is close-ended and low-ambiguity (binary safety label, format-compliance yes/no), two annotators is usually sufficient. If the rubric has multiple gradations and requires judgment (a 4-point quality scale with anchors), a third annotator on the items where the first two disagree — a *tiebreak* rater — meaningfully improves gold quality.

## Which agreement statistic

The choice depends on the rubric's output shape.

**Categorical labels (binary or small-ordinal, unordered).** Use **Cohen's κ** between the judge and adjudicated human labels.
- `sklearn.metrics.cohen_kappa_score(human, judge)`.
- κ ranges from -1 (systematic disagreement) to +1 (perfect agreement), with 0 meaning "no better than chance given the label distribution."
- For > 2 raters or > 2 labels, **Fleiss's κ** or **Krippendorff's α** generalize; on the two-way judge-vs-adjudicated-human case Cohen's κ is standard.

**Ordinal labels (ordered categories or Likert scale).** Use **weighted κ** (quadratic weights) instead of plain κ. Weighted κ penalizes larger disagreements more than adjacent ones — a judge that says "4" when a human said "3" is much closer to the truth than a judge that says "1", and plain κ does not distinguish those cases.

**Continuous or ranking scores.** Use **Spearman's ρ** or **Kendall's τ** between the judge score and the human score.
- Spearman is a Pearson correlation on the ranks of the two scores; robust to monotonic-but-non-linear relationships.
- Kendall's τ counts concordant vs discordant pairs; more interpretable ("fraction of pairs the judge ranks in the same order humans do") but more expensive on large sets. Use τ-b when there are ties.
- Both live in `scipy.stats`.

For pairwise judging where the judge output is a preference (A / B / tie) and the human gold is also a preference, the natural agreement statistic is *accuracy* (fraction of pairs where the judge agrees with the human), sometimes also κ if the tie rate is meaningful.

**Do not use raw accuracy without also reporting κ on categorical rubrics.** Cohen's κ is prevalence-sensitive but so is your intuition: on a rubric where 90% of items are "safe," a judge that always says "safe" has 90% accuracy and κ ≈ 0. Accuracy alone will lie about that judge; κ will not.

## Sizing the study: what agreement can you resolve?

The bootstrap CI on κ (or ρ, or τ) shrinks with sample size, and there is a specific inflection: below about 30–50 items, the CI is wide enough that "κ = 0.7" and "κ = 0.4" are statistically indistinguishable, so any decision you draw from the number is unfounded. Practical anchors:

- **30–50 items** — sufficient only for a *sanity check* on a low-stakes internal tool. CIs are wide.
- **100–200 items** — the sweet spot for most calibration studies. κ CI of ±0.05–0.10 depending on the label distribution. Enough to make a ship decision.
- **500+ items** — required if you need to resolve κ differences of ~0.03, or if you want per-slice calibration (κ per topic, per length bucket, per user segment).

Any calibration study on fewer than 30 items and any comparison of two judges' κ without an overlap CI is measurement theater. Report the CI on the agreement statistic with bootstrap resamples (mod-101 Chapter 2 for the method).

## Interpreting the numbers

The following bands are a starting point, adapted from Landis and Koch 1977 and refined by consistent use in the model-graded eval literature (Zheng et al. 2023 uses κ ≥ 0.6 as a working threshold for "judge is aligned enough with humans to trust"; OpenAI evals' rubric-validation guidance uses ≥ 0.7). Treat them as decision anchors, not natural constants — the actual threshold depends on the *decision stakes* and on *human-human agreement on the same rubric*.

- **κ ≥ 0.8** — near-human agreement. Judge is safe to use as a metric in most contexts. On many rubrics, humans do not agree with each other this well; a judge that reaches this level is either measuring a very close-ended task or the rubric is simple enough that both humans and judge converge easily.
- **0.6 ≤ κ < 0.8** — substantial agreement. Ship-able for a monitorable metric or a diff-alerting signal. Not sufficient for a metric that gates a launch or a paycheck without additional caveats.
- **0.4 ≤ κ < 0.6** — moderate agreement. The judge is measuring *something* correlated with the human signal, but there is a lot of noise. Usable only for coarse ranking (system A vs system B is a large enough difference to survive the noise) and only with an explicit caveat about the calibration level.
- **κ < 0.4** — the judge and humans disagree too often to trust the judge as a metric. Do not ship. The rubric, the judge, or both need rework.

The critical caveat: **compare judge-human κ to human-human κ on the same rubric**, not to an absolute threshold. If humans only agree with each other at κ = 0.5 on your task (some subjective rubric), then a judge at κ = 0.5 is *as reliable as a human rater*, and asking for κ ≥ 0.7 makes no sense. Always compute human-human κ from your annotator pairs and report it alongside judge-human κ. The judge should be roughly as good as an *average human rater*, not better than a rater panel.

## The confusion matrix is where you learn what to fix

The agreement statistic tells you *whether* the judge is aligned with humans. The confusion matrix tells you *where it is not* — and that is what you use to decide whether the fix is a rubric edit, a judge swap, or a task descope.

For a 4-point ordinal rubric, a `4 × 4` confusion matrix of judge label vs human label reveals patterns like:

- **Off-diagonal but adjacent** — the judge is one score point off systematically. Often a rubric anchor problem (the judge's interpretation of "3" is your interpretation of "4"). Fix: sharpen the anchor descriptions on those two adjacent cells; rerun.
- **Off-diagonal and non-adjacent** — the judge is confusing categories that a human would not. Often a judge-capability problem or a rubric-criteria problem (the rubric asks about "correctness" but the judge is scoring "helpfulness"). Fix: revisit the criteria wording; consider a stronger judge.
- **Asymmetric errors** — the judge over-scores in one direction. Common with length bias (judge over-scores long responses) or self-preference (judge over-scores its own family). Fix: apply Chapter 4's controls; re-measure.

Every calibration report should include the confusion matrix, not just the summary statistic.

## The ship decision

The end state of a calibration study is a decision. Structure the decision as a checklist, not as a single-number gate.

1. **Is judge-human agreement above the ship threshold for the decision this metric supports?** Threshold depends on stakes. Regression-alerting dashboards can ship at κ ≥ 0.5; launch-gating metrics need κ ≥ 0.7 or a human-in-the-loop backstop.
2. **Is judge-human agreement within 0.1 of human-human agreement on the same rubric?** If humans only agree at κ = 0.55, a judge at κ = 0.50 is measuring the ceiling. If judge is much worse than humans, the *judge* is the limitation and rework is worthwhile.
3. **Are the systematic errors documented?** The confusion matrix has been read, the modes of disagreement have been named, and either a fix or an explicit caveat is in the report.
4. **Are the biases from Chapter 4 controlled and reported?** Position, length, self-preference. A high κ that fell out of a length-biased judge on a length-uniform test set will collapse the first time the deployed data has variable-length responses.
5. **Is there a plan to re-measure?** Calibration is not a one-time event. The judge model version will change (silent updates on hosted APIs are common), the data distribution will drift, and the rubric may be revised. The re-measurement cadence — monthly, quarterly, on-every-judge-version-bump — is a plan, not a hope.

If all five are green, ship. If any are red, the specific red is what you fix; do not average the greens against the red.

## When calibration keeps failing

If two rubric iterations and a stronger judge still leave you below the ship threshold, the diagnosis is usually one of the following, in decreasing order of frequency:

- **The task is genuinely harder than the rubric assumes** — the criteria involve judgment that experts do not agree on. This is a task-descope decision, not a judge decision. Break the rubric into narrower sub-criteria that experts *do* agree on, and calibrate each separately.
- **The reference answers used in the rubric are wrong or ambiguous.** Judges can only be as consistent as the reference. Audit 20 low-agreement items; if the reference is at fault on > 5, fix the references before touching the judge.
- **The label distribution is too skewed** — 90% of items are one label. κ under skew is a trap (mentioned above). Rebalance the calibration set to include enough of the minority label for κ to be well-defined.
- **The judge model is genuinely not capable enough for the task** — this is diagnosable by running the same rubric with a stronger judge (frontier vs mid-tier). If κ jumps by > 0.15, the judge tier is the bottleneck and Chapter 7's routing question is next.

## Summary

Calibrating a judge against humans is the study that turns a plausible number into a measurement. The mechanics: 50–200 items, at least two independent human annotators with a written adjudication rule, judge labels produced with the same configuration you will run in production, and one of Cohen's κ (categorical), weighted κ (ordinal), or Spearman's ρ / Kendall's τ (continuous / ranking) computed with a bootstrap CI. The interpretation: κ compared against both a decision-stakes threshold and against human-human κ on the same rubric, plus a confusion matrix read for systematic disagreement modes. The ship decision is a five-check gate, not a single-number bar. The next chapter shifts from calibrating a single judge to aggregating many pairwise verdicts into a system rating — the Bradley-Terry and ELO machinery behind arena-style leaderboards.
