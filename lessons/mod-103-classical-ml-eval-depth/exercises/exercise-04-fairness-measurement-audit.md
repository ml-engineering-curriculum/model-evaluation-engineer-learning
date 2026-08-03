# exercise-04: Fairness Measurement Audit

**Estimated effort:** 3 hours

## Objective

Run an end-to-end fairness audit on a real classifier and dataset with a protected attribute. Compute the standard fairness metrics from Chapter 5 using both Fairlearn and Aequitas, report the trade-offs the impossibility theorem forces, and write a policy-owner-ready audit memo.

The point is not to satisfy every fairness definition — the impossibility results say you often cannot when base rates differ. The point is to name the trade-off explicitly, ground it in evidence, and produce the artifact a legal/policy reviewer would sign off on.

## Prerequisites

- mod-103 Chapters 5 and 6.
- mod-101 Chapters 3–4 (Wilson and bootstrap CIs).
- mod-103 Chapter 1 (per-slice reporting) and Chapter 4 (operating points).
- Python 3.11+ with `fairlearn`, `aequitas`, `scikit-learn`, `pandas`, `numpy`, `matplotlib`.

## Datasets (pick one)

Each dataset has a documented protected attribute and a well-known set of fairness papers to compare against. Use one of these; do not roll your own synthetic dataset for this exercise — the point is to work with real, imperfect protected-attribute data.

- **UCI Adult / Census Income.** Protected attributes: `sex`, `race`. Standard baseline in the fairness literature; used in Hardt et al. (2016) and dozens of follow-ups.
- **COMPAS recidivism (ProPublica).** Protected attributes: `race`, `sex`, `age_cat`. The dataset that motivated the impossibility papers (Chouldechova 2017; Kleinberg et al. 2017). Cite the ethical and statistical criticism when you use it.
- **HMDA public loan data (US CFPB, most recent annual release).** Protected attributes: `applicant_race`, `applicant_sex`, `applicant_ethnicity`. Real regulatory data with the fairness axes defined in the disclosure schema itself. The most realistic environment for this exercise; also the largest, so budget time for the download.
- **German Credit (UCI Statlog).** Protected attribute: `personal_status` and (derived) `age`. Small (`n = 1000`), so the CIs will be wide — which is a legitimate finding, not a bug.

You may substitute an internal dataset you are allowed to write about; document the protected attribute's source and any measurement error (self-report, inference, proxy).

## Requirements

### Part A — the audit scope memo

Before touching the model, write `AUDIT_SCOPE.md` (~300–500 words). It must include:

1. **The classifier's purpose.** What decision does its output drive; what would a positive prediction lead to; who acts on it.
2. **The protected attribute(s).** How were they collected (self-reported, inferred, external)? What is their known measurement error?
3. **The reference group.** Which group is the reference for difference/ratio metrics, and why (regulatory, largest group, policy choice).
4. **The primary fairness definition.** Which one — demographic parity, equalized odds, equal opportunity, predictive parity, or calibration-within-groups — is the primary metric, and *why*? Argue against the alternatives. This is the decision you make before you see the numbers.
5. **The intersections to report.** Which two-way intersections (e.g. race × sex) will be reported, subject to sparse-slice suppression.
6. **The accountable owner** of the audit. Who signs off on the final memo; who is empowered to act on the recommendations.

### Part B — train (or load) the classifier

Train a baseline binary classifier — logistic regression or a boosted tree is plenty. Fix the operating point using Chapter 4's discipline; document what threshold you are auditing at and why. If you did exercise-01 or exercise-03 on the same dataset, reuse the model.

Save `y_true`, `y_pred` (thresholded), `y_score`, and the protected-attribute column(s) on the test set.

### Part C — the Fairlearn measurement

Use `fairlearn.metrics.MetricFrame` to compute the per-group metrics.

Required outputs:

1. **Per-group table** with columns: `group`, `n`, `selection_rate` (for DP), `tpr` (for EO / EqOpp), `fpr` (for EO), `precision` (for PP), `accuracy`. Save as `fairness_group_metrics.csv`.
2. **Summary metrics** using Fairlearn's helpers:
   - `demographic_parity_difference` and `demographic_parity_ratio`.
   - `equalized_odds_difference` and `equalized_odds_ratio`.
   - `equal_opportunity_difference` (compute manually: `max_a TPR_a − min_a TPR_a`).
   - Per-group precision — the predictive-parity diagnostic.
3. **Per-group calibration** — for each group, compute ECE (Chapter 2). Save the reliability diagrams for each group on the same axes as `fairness_reliability.png`.
4. **CIs on every metric.** Bootstrap the whole audit `B = 1000` times, resampling rows of the test set (stratified by group to keep small groups from disappearing). Report 95% CIs on every summary metric.

### Part D — the Aequitas audit

Format the test-set predictions as an Aequitas-compatible dataframe (`label_value`, `score`, and one column per attribute). Run:

1. `Group.get_crosstabs(df)` — per-group counts and rates.
2. `Bias.get_disparity_predefined_groups(..., ref_groups_dict={...}, alpha=0.05)` — bias metrics with respect to your reference group.
3. `Fairness.get_group_value_fairness(...)` — fairness determinations at the tolerance you specify (e.g. four-fifths rule for demographic parity ratio).

Save the resulting tables as `aequitas_bias.csv` and `aequitas_fairness.csv`. Compare against your Fairlearn numbers — they should agree closely; if they do not, describe the discrepancy (usually a definition or reference-group difference).

### Part E — the impossibility trade-off analysis

This is the load-bearing part of the audit.

1. **Base rates by group.** Compute `P(Y = 1 | A = a)` for each protected group. If the base rates differ non-trivially, the impossibility theorem applies.
2. **The three-way trade-off table.** Report per group and the disparity across groups:
   - Predictive parity (PPV per group).
   - Equalized-odds decomposition (TPR and FPR per group).
   - Calibration within groups (ECE per group).
   
   Identify which pair (or more) cannot be simultaneously satisfied given the observed base-rate divergence.
3. **Discuss which of the incompatible definitions your organization is prioritizing** based on the primary metric chosen in Part A.
4. **Do the impossibility numerics.** Cite Chouldechova (2017) or Kleinberg et al. (2017) as the reason the trade-off is unavoidable — do not present it as a modelling failure. Show the base-rate gap explicitly.

### Part F — intersectional analysis

For the two most-important protected attributes, compute per-intersection metrics (e.g. race × sex). Report:

- `n_intersection` for each cell.
- Selection rate, TPR, FPR, precision, ECE per cell.
- Cells with `n < 30` (or your documented threshold) flagged as sparse; do not compute per-cell disparities on sparse cells.
- The largest intersectional gap on the primary fairness metric.

Cite Buolamwini & Gebru (2018) — the finding that single-attribute audits can hide intersectional harms.

### Part G — the fairness audit memo

Write `FAIRNESS_AUDIT.md` (~800–1200 words) suitable for a legal or policy reviewer. Structure:

1. **One-sentence finding.** "The model satisfies [primary metric] within [tolerance] and fails [other metric] with disparity [X]."
2. **Scope recap** — dataset, classifier, operating point, protected attribute, reference group, primary fairness metric.
3. **Per-group table** (from Part C) with CIs.
4. **Trade-off discussion** — which definitions could be satisfied; which are in tension; why the primary was chosen.
5. **Intersectional finding.**
6. **Recommended action** — one of: ship / ship with conditions / do not ship / mitigate before shipping. If mitigation, name the specific mitigation and the trade-off it introduces (e.g. per-group thresholds → policy question).
7. **What to monitor after ship** — the metric that will drift first if group base rates change.
8. **Accountable owner** and next-review date.

## Starter guidance

- Do not skip Part A. A fairness audit whose primary metric was chosen after the numbers were computed is not defensible.
- For UCI Adult with `sex` as the protected attribute, expect demographic-parity ratio ~0.4 with a logistic regression at default threshold — the disparity is large and well-documented. Treat this as validation that your pipeline works.
- Fairlearn's `MetricFrame.difference()` returns the max group difference by default; `MetricFrame.ratio()` returns min/max. Both are reasonable; pick one and document.
- Aequitas's `Fairness.get_group_value_fairness` requires you to pick tolerances (e.g. `fair_threshold = 0.8` for the four-fifths rule on ratio metrics). State your tolerances and their source.
- Bootstrap CIs on fairness metrics: sample rows with replacement, recompute the summary metrics, take percentiles. For small groups, use a *stratified* bootstrap that resamples within each group to keep group sizes stable. Otherwise a tiny group can vanish in some resamples and inflate variance.
- For the intersectional cells, do not silently pool sparse cells into `other`; report them explicitly with the sparse flag.
- The COMPAS dataset in particular: cite Chouldechova (2017) and both the ProPublica and Northpointe positions in the trade-off discussion. Do not take a side; report the numbers and the choice.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- `AUDIT_SCOPE.md` names the protected attribute (with its collection method and error properties), the reference group, the primary fairness metric, and the accountable owner — before any numbers are computed.
- Both Fairlearn and Aequitas are used and their outputs agree (or the disagreement is diagnosed and explained).
- Every fairness metric has a 95% bootstrap CI.
- The impossibility trade-off is discussed explicitly with reference to Chouldechova (2017) or Kleinberg et al. (2017); base rates by group are reported.
- Intersectional analysis is present for at least one two-way intersection, with sparse-cell suppression flagged where relevant.
- `FAIRNESS_AUDIT.md` fits the requested structure, opens with a one-sentence finding, and names the accountable owner.
- The pipeline reproduces deterministically from a single script.

## Stretch goals

- **Mitigation and re-audit.** Apply a Fairlearn mitigator (`ThresholdOptimizer` for post-processing, or `ExponentiatedGradient` with `DemographicParity` or `EqualizedOdds` for in-processing). Re-run the entire audit on the mitigated model. Report the trade curve of accuracy vs. disparity — and discuss whether the mitigation moves the disparity to a different definition or a different intersection.
- **Sensitivity to threshold.** Sweep the classifier threshold across `[0.1, 0.9]` and plot the disparity in your primary fairness metric vs. threshold. Discuss how the operating-point choice (exercise-03) interacts with the fairness picture.
- **Inferred-attribute audit.** If your dataset supplies a self-reported protected attribute, also compute an inferred version (e.g. BISG surname-based race estimate for a subset with available surname data). Compare the audit result under the two attribute sources; discuss what the difference implies about inferred-attribute fairness reporting.
- **Predictive parity vs. equalized odds comparison.** For your dataset, compute both. Confirm the arithmetic-level impossibility numerically: no threshold makes both hold. Show a two-axis chart of `PPV disparity` vs. `EO disparity` for a sweep of thresholds; identify the pareto frontier.
- **Calibration-in-groups follow-up.** If per-group calibration is meaningfully different, fit per-group Platt or isotonic calibrators (Chapter 3) and re-audit. Report whether per-group calibration reduces the primary disparity, and discuss the policy trade-off of per-group calibrators.
