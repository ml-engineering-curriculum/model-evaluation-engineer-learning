# exercise-03: Operating Point Selection Under Cost Asymmetry

**Estimated effort:** 2 hours

## Objective

Take a classifier plus a plausibly-motivated cost story, and pick an operating point three different ways: expected-utility maximization, Neyman–Pearson (constrained), and fixed-budget. Report each with the confusion matrix, CIs, a sensitivity analysis on the cost ratio, and a per-slice breakdown. Produce the shipping memo a product reviewer would sign off on.

The point is to make the operating-point choice explicit and defensible — not to default to `0.5` and hope the numbers land somewhere reasonable.

## Prerequisites

- mod-103 Chapters 2, 3, and 4.
- mod-101 Chapter 4 (bootstrap CIs).
- Python 3.11+ with `numpy`, `pandas`, `scikit-learn`, `matplotlib`.

## Scenarios (pick one)

Each scenario supplies a dataset, a classifier suggestion, and a written cost story. Substitute your own numbers where they say "your organization decides"; the exercise is about the *reasoning*, not memorizing a specific business case.

### Scenario A — fraud triage (recommended default)

- **Dataset.** Credit-card fraud dataset (Dal Pozzolo et al., available on Kaggle as `mlg-ulb/creditcardfraud` or via `sklearn.datasets.fetch_openml("CreditCardFraudDetection")` variants). Extreme class imbalance (~0.17% positives).
- **Model.** Any calibrated binary classifier (logistic regression + Platt, or a gradient-boosted tree). Reuse your model from exercise-02 if you did it.
- **Cost story.** A **false positive** blocks a legitimate transaction, causing customer frustration and downstream support cost — assume `c_FP = $5` in customer-goodwill cost. A **false negative** lets a fraudulent transaction through — assume `c_FN = $250` (median disputed-amount * probability of chargeback). `c_TP` and `c_TN` are `$0`. Your CFO revises `c_FN` between `$50` and `$500` monthly; treat the range as the sensitivity band.
- **Budget constraint (for framing 3).** The fraud-review team can review at most 1200 flagged transactions per day; the eval set corresponds to one day of traffic.

### Scenario B — medical screening

- **Dataset.** Wisconsin Breast Cancer diagnostic dataset (`sklearn.datasets.load_breast_cancer`) — small, well-behaved, canonical. Or a diabetes-onset prediction dataset (Pima Indians, on Kaggle) for a larger `n`.
- **Model.** Any calibrated binary classifier.
- **Cost story.** A **false positive** triggers a follow-up test (biopsy, additional imaging) — assume `c_FP = $500`. A **false negative** delays diagnosis — assume `c_FN = $50000` (a stand-in for the expected difference in outcome-adjusted cost of late vs. early detection; the point is the ratio, not the absolute number). Ask the clinical lead to specify the range on `c_FN` you should sensitivity-test over.
- **Constraint (for framing 2).** Public health guidance in your jurisdiction requires screening tests to achieve sensitivity (recall on the positive class) ≥ 0.95. Your Neyman–Pearson formulation is: maximize precision subject to this.

### Scenario C — content moderation flag-for-review

- **Dataset.** A public content-classification dataset — Jigsaw Toxic Comment Classification (Kaggle, Wikipedia comments) or a similar labelled corpus.
- **Model.** Any calibrated classifier.
- **Cost story.** A **false positive** (benign content flagged) delays legitimate posting and may propagate to a moderator's queue — `c_FP = $0.50` in reviewer time. A **false negative** (harmful content missed) has policy and reputational cost that varies wildly by severity — pick a plausible `c_FN` range (say $10 to $1000) and treat the choice as the load-bearing product decision.
- **Constraint (for framing 2).** Trust-and-safety policy requires false-positive rate on non-toxic content to be at most 3%.
- **Budget (for framing 3).** The human-review team processes at most 2000 flagged posts per day.

## Requirements

### Part A — the cost story writeup

Write `COST_STORY.md`, ~200–400 words. Before touching numbers:

1. Name the scenario and the classifier's role.
2. State `(c_TP, c_FP, c_TN, c_FN)` with your source for each.
3. Name the accountable owner of the cost numbers — a plausible role (Fraud Head, Trust & Safety Director, Clinical Lead), not just "the model owner."
4. State the sensitivity range you will test `c_FN / c_FP` over.
5. State the Neyman–Pearson constraint and the budget for the alternative framings.

The point of writing this before the computation is to prevent "backing into" a cost ratio that produces a threshold you already wanted.

### Part B — the classifier and its calibration status

Train (or reuse) a binary classifier. Report:

- Its aggregate accuracy, precision, recall, F1, and AUC on the test set with CIs.
- Its ECE (adaptive-binned, 15 bins) with a CI (Chapter 2).
- Whether it has been recalibrated and by which method (Chapter 3). If uncalibrated, note this — the Bayes-optimal closed-form threshold does not apply.

Save `y_true`, `y_score`, and the slice columns for downstream use.

### Part C — three operating-point choices

Implement the three framings from Chapter 4 in `operating_points.py`.

1. **Expected-utility minimization.** Enumerate candidate thresholds from the sorted unique scores; compute per-threshold expected cost with `(c_TP, c_FP, c_TN, c_FN)`; take the argmin. Report the chosen threshold `t*_EU`, the confusion matrix at `t*_EU`, the expected cost with a bootstrap CI, and where `t*_EU` sits relative to the Bayes-optimal closed-form threshold (they should agree closely if the model is well-calibrated).
2. **Neyman–Pearson.** Take the constraint from the cost story (Scenario A: precision ≥ some target; Scenario B: recall ≥ 0.95; Scenario C: FPR ≤ 0.03). Pick the threshold that maximizes the other axis subject to the constraint. Report `t*_NP`, the achieved constrained axis, and the maximized axis, all with CIs.
3. **Fixed budget.** Pick `t*_B` such that the number of positive predictions on the eval set equals the daily budget. Report `t*_B`, the resulting expected precision, and how it moves if the budget shifts by ±10%.

### Part D — the sensitivity analysis

For Part C.1 (expected utility):

1. Sweep `c_FN` across the sensitivity range from `COST_STORY.md` while holding `c_FP` fixed. Plot `t*_EU` vs. `c_FN / c_FP` as a line chart. Include the current best-guess ratio as a vertical rule.
2. Report the elasticity: for a 10% change in `c_FN / c_FP`, how much does `t*_EU` move (in raw score units, and in percentile-of-score-distribution units)? If the elasticity is high, `t*_EU` is fragile.
3. Recompute the confusion matrix and expected cost at the ±20% cost ratios and show whether the recommendation changes.

### Part E — per-slice operating points

Pick 2–3 slicing axes from your dataset (reuse exercise-01 if applicable). For each of the three operating-point choices in Part C:

1. Apply the *global* threshold to each slice; report per-slice precision, recall, FPR, and expected cost with CIs.
2. Discuss which slice bears the greatest share of the false-positive rate and which bears the greatest false-negative rate. Are the slices where the model does worst also the slices where the cost is highest?
3. (Optional but recommended) Compute what the *per-slice* Bayes-optimal thresholds would be. Do not silently ship them — Chapter 5 covers why per-slice thresholds are a fairness decision that requires explicit justification.

### Part F — the shipping memo

Write `SHIPPING_MEMO.md` (~600 words). Structure:

1. **Recommendation** — one sentence at the top: "Ship at threshold `t = X` under framing [expected-utility / Neyman–Pearson / fixed-budget]."
2. **Cost story recap** — one paragraph.
3. **Chosen framing and why** — argue against the two you did not choose.
4. **Numbers at the chosen threshold** — the confusion matrix, precision, recall, FPR, expected cost, all with CIs.
5. **Sensitivity finding** — one sentence on how fragile the threshold is to the cost ratio.
6. **Per-slice picture** — the two or three worst-off slices at the chosen threshold and what to do about them.
7. **What to monitor after ship** — the metric that will move first if the cost ratio drifts or the population shifts.

## Starter guidance

- Use `sklearn.metrics.precision_recall_curve` and `roc_curve` to get pre-sorted (`threshold`, `precision`, `recall`, `fpr`, `tpr`) arrays. These are ~30x faster than looping over candidate thresholds by hand.
- Bootstrap the *threshold choice* — resample the eval set with replacement `B=1000` times, rerun the choice procedure on each resample, take the 2.5 / 97.5 percentiles of the chosen threshold. A wide CI here means the operating point is not well-identified.
- For fraud data with 0.17% positives, the ROC curve is misleading (Chapter 4). Use the PR curve.
- If your classifier is uncalibrated, note that the *Bayes-optimal closed-form* threshold does not directly apply — the expected-cost sweep still works, but the ratio-to-threshold intuition breaks. Recalibrate first if you can.
- The Neyman–Pearson choice is often more stable than expected-utility because it does not require the cost ratio; use the stability difference as an argument in the memo when applicable.
- For the per-slice discussion, do not fit per-slice thresholds without explicitly acknowledging that this is what fairness Chapters 5–6 are about.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- `COST_STORY.md` names `(c_TP, c_FP, c_TN, c_FN)`, the sensitivity range, and the accountable owner of the cost numbers — before any operating-point computation.
- All three framings (expected-utility, Neyman–Pearson, fixed-budget) are implemented and produce distinct operating points.
- Every reported threshold has a bootstrap CI, and every reported confusion-matrix-derived metric has a bootstrap CI.
- The sensitivity analysis includes a plot of `t*` vs. `c_FN / c_FP` and reports the elasticity, plus what happens at the ±20% ratios.
- The per-slice section reports the slice(s) that bear the greatest share of FP and FN at the chosen operating point.
- `SHIPPING_MEMO.md` fits on one page, opens with a one-sentence recommendation, argues against the alternative framings, and includes a monitoring recommendation.
- The pipeline reproduces from a single script given the seed and the scenario choice.

## Stretch goals

- **Calibration cross-check.** Report the same three operating points using both uncalibrated and calibrated scores; discuss whether the choice moves in a way consistent with Chapter 3's expectation.
- **Per-slice Neyman–Pearson.** Find per-slice thresholds that satisfy the FPR ≤ 3% (or recall ≥ 0.95) constraint slice-by-slice. Report the disparity in the maximized axis across slices, and write an explicit fairness note (Chapter 5) on whether that disparity is defensible.
- **Threshold randomization.** For an equalized-odds-style requirement on the operating point, implement the Hardt–Price–Srebro (2016) randomized threshold. Report the resulting per-group operating points and their randomization probabilities. Discuss why stochastic decisions are often unacceptable in high-stakes settings.
- **Cost-curve visualization.** Plot expected cost vs. threshold across the whole `[0, 1]` range and mark all three chosen thresholds. Discuss the shape (single minimum? plateau? multi-modal?).
- **Cross-ratio robustness.** For the Neyman–Pearson choice, compute how much your maximized axis (e.g. recall at FPR ≤ 0.03) would change if the constraint were tightened to 0.02 or loosened to 0.05. This is the alternative sensitivity analysis when the cost ratio is not available.
