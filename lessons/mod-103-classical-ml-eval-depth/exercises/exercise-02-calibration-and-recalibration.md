# exercise-02: Calibration and Recalibration

**Estimated effort:** 3 hours

## Objective

Take a classifier whose raw scores are likely miscalibrated, measure the miscalibration with a reliability diagram and ECE / MCE / Brier, then fit Platt scaling, isotonic regression, and temperature scaling on a held-out calibration split. Compare all three on a fresh test split with all the diagnostics from Chapter 2, and pick a winner with a written justification.

The point is not to get one specific answer — different datasets and different classifier families genuinely favor different calibrators — but to make the picking-a-calibrator decision on evidence rather than convention.

## Prerequisites

- mod-103 Chapters 2 and 3.
- mod-101 Chapter 4 (bootstrap CIs).
- Python 3.11+ with `numpy`, `pandas`, `scikit-learn`, `matplotlib`, and (for temperature scaling) `torch` if the model you pick has raw logits.

## Datasets and models (pick one row from the table below)

| Dataset | Model | Why this pair miscalibrates |
|---|---|---|
| UCI Adult (sklearn `fetch_openml("adult", version=2)`) | `GaussianNB` from sklearn | Naive Bayes is notoriously overconfident. |
| CIFAR-10 (`torchvision.datasets.CIFAR10`) | Small ResNet-18 trained a few epochs to convergence | Modern deep nets are systematically overconfident (Guo et al. 2017). |
| Bank Marketing (UCI) | `RandomForestClassifier(n_estimators=100)` | Random forests are sigmoid-miscalibrated (Niculescu-Mizil & Caruana 2005). |
| Fashion-MNIST | Any well-trained multiclass classifier of your choice | Multiclass calibration exercises top-label vs. classwise. |
| Any binary classification dataset ≥ 10K rows | Linear SVM (`sklearn.svm.LinearSVC` with `decision_function`) | SVM margins are not probabilities — Platt was invented for this. |

You may substitute a real internal model you are allowed to write about; the point is one that miscalibrates in a specific documentable way.

## Requirements

### Part A — the calibration measurement (before recalibration)

Split the data three ways: **train** (~70%), **calibration** (~15%), **test** (~15%). Stratify on the label. Fix and document the seed. Never touch the test split until the final evaluation.

Train the base model on the train split. Produce raw scores on the test split. Then:

1. **Reliability diagram** with both fixed-width bins (10 bins) and adaptive / quantile bins (15 bins), rendered side by side. Overlay the score histogram. Save as `reliability_before.png`.
2. **Compute ECE, MCE, and Brier score** on the test split, each with a 95% bootstrap CI (`n_boot = 1000`). For a binary classifier use the standard formulas from Chapter 2; for multiclass use top-label calibration by default, and classwise calibration as a stretch goal.
3. **Per-slice ECE** on 2–3 slicing axes from your dataset (reuse the axes from exercise-01 if you did it). Flag any slice whose ECE exceeds 2× the aggregate.
4. Write `CALIBRATION_BEFORE.md` — ≤ 300 words. What does the reliability diagram look like? Where is the model over- or under-confident? Is there a slice-specific miscalibration problem? Does the model need recalibration, or does it pass a "well enough" bar?

### Part B — fit the three calibrators

On the held-out calibration split (not the training set, not the test set):

1. **Platt scaling.** One-dimensional logistic regression on the raw score (or margin). `sklearn.linear_model.LogisticRegression` fitted to `(raw_cal.reshape(-1, 1), y_cal)`.
2. **Isotonic regression.** `sklearn.isotonic.IsotonicRegression(out_of_bounds="clip")` fitted to `(raw_cal, y_cal)`.
3. **Temperature scaling.** Only if you have raw logits. Fit a single scalar `T > 0` by minimizing NLL on `(logits_cal, y_cal)`. LBFGS in `torch` is the standard implementation; a plain SciPy `minimize_scalar` also works.

If you cannot fit temperature scaling (no logits available), replace it with **beta calibration** (`betacal.BetaCalibration`) or with `sklearn.calibration.CalibratedClassifierCV(base, method="sigmoid")` used with cross-validation — document the substitution.

### Part C — compare on the test split

Apply each calibrator to the raw scores on the test split. Produce:

1. **Reliability diagrams** for uncalibrated + all three calibrators, on the same axes, saved as `reliability_after.png`.
2. **Metrics table.** For each of {uncalibrated, Platt, isotonic, temperature (or beta)}, compute:
   - ECE (adaptive-binned, 15 bins) with CI.
   - MCE with CI.
   - Brier score with CI.
   - Log loss with CI.
   - Accuracy at threshold `0.5`.
   - AUC.

   All with 95% bootstrap CIs (`n_boot = 1000`).
3. **Per-slice ECE.** Report per-slice ECE for the best global calibrator and compare against the pre-calibration per-slice ECE from Part A. Did recalibration fix the worst slice or shift the problem elsewhere?

### Part D — the choice memo

Write `CALIBRATION_MEMO.md` (~400–600 words). It must contain:

1. **Which calibrator won**, with a table like the one in Chapter 3 that presents the numbers side by side.
2. **Justification for the choice.** If two calibrators tie on the primary metric, name the tiebreaker (prefer lower capacity, prefer preserved accuracy, prefer better slice-level fix). Do not choose by feel.
3. **What can go wrong at production time.** If the production base rate differs from the calibration-set base rate, what happens to the calibrator? If the input distribution shifts, what monitoring signals would tell you the calibrator has degraded? What is the recalibration cadence you would recommend?
4. **Recommendation:** ship recalibrated / ship uncalibrated / do not ship. Argue against the alternatives.

### Part E — a small pitfall check

Include the following "did I fall for it?" checks explicitly in the writeup:

1. Did the temperature-scaled model preserve accuracy exactly? (It must, by construction.) If not, there is a bug in the pipeline.
2. Does the isotonic-calibrated model produce a strange distribution of probabilities (piecewise-constant "plateaus")? Show a histogram of `p̂_iso` on the test set and comment.
3. Does any calibrator move AUC materially (> a bootstrap-noise amount)? If so, describe why — usually a hint that the calibrator overfit or violated monotonicity.

## Starter guidance

- Use `sklearn.calibration.calibration_curve` with `strategy="quantile"` for the adaptive bins; the fixed-width version is `strategy="uniform"`.
- For the score histogram overlay, use a secondary y-axis or a smaller sub-panel; do not let the histogram bars dominate the reliability points visually.
- `sklearn.calibration.CalibratedClassifierCV` implements the cross-validated version of Platt and isotonic. It is a fine alternative when you cannot spare a fresh calibration split, but be explicit about which choice you made and why.
- Temperature scaling with `torch`: one parameter, one LBFGS call, one loss (NLL). ~15 lines total. See Guo et al. (2017) supplementary material for the reference implementation.
- For Naive Bayes on Adult, expect ECE > 0.15 before recalibration and < 0.03 after. If your numbers are much better or much worse, look for a bug.
- Bootstrap CIs for ECE: resample rows of `(y_true, y_score)` with replacement, recompute ECE on each resample, take the 2.5 / 97.5 percentiles. The distribution can be skewed; do not assume a Gaussian CI.
- For per-slice ECE: pick slices with `n_slice ≥ 200`. Smaller slices have ECE dominated by binning noise and the report becomes uninterpretable.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- The train / calibration / test three-way split is documented, reproducible, and the test split is untouched until the final evaluation.
- `reliability_before.png` shows both binning strategies with the score histogram overlaid.
- ECE / MCE / Brier / log loss are reported for uncalibrated + all three calibrators with 95% bootstrap CIs.
- The reliability diagram in `reliability_after.png` shows all four curves overlaid so the shape improvement is visible.
- Per-slice ECE is computed and the report says whether recalibration fixed, worsened, or shifted the per-slice picture.
- `CALIBRATION_MEMO.md` picks a calibrator on evidence (not convention), names the tiebreaker if applicable, and includes a production-monitoring recommendation.
- The Part E "did I fall for it" checks are addressed and any anomalies are explained.
- The pipeline re-runs deterministically from a single script given only the seed and the dataset choice.

## Stretch goals

- **Debiased ECE (Kumar et al. 2019).** Implement the kernel or debiased-binning ECE estimator and report it alongside the standard binned ECE. Discuss whether it changes the ranking of the calibrators.
- **Per-slice calibrators.** Fit a separate Platt / isotonic calibrator per slice for the axes where global calibration is still poor. Compare against the global calibrator. Discuss why slice-specific calibration is not always desirable (it is a form of per-group threshold — see Chapter 5).
- **Classwise multiclass calibration.** For a multi-class model, report both top-label ECE and per-class ECE. Show a class-wise reliability grid.
- **Time-lagged recalibration.** If your dataset has a time column, split so calibration is on one time window and test is on a later one. Report the calibration drift and use it as evidence for a recommended recalibration cadence.
- **Baseline against `CalibratedClassifierCV`.** Compare your hand-fit calibrators against `sklearn.calibration.CalibratedClassifierCV` with `cv=5`. If they differ materially, explain (usually the CV version smooths the calibrator).
