# Recalibration: Platt Scaling, Isotonic Regression, and Temperature Scaling

Chapter 2 showed how to detect miscalibration. This chapter is what to do about it. The good news is that recalibration is almost free — it is a small post-hoc model fit on a held-out calibration set that maps the raw model scores to well-behaved probabilities without retraining the underlying model. The three standard methods differ in how much structure they assume, how much calibration data they need, and how they behave at the extremes of the score range. Picking between them is a small design decision, and it is one you should make on evidence rather than habit.

## The recalibration setup

You have:

- A trained scoring model that outputs a raw score `s(x)` — a logit, a decision-function value, a probability estimate, or a margin.
- A **calibration set** disjoint from both the training set and the test set. This is the load-bearing requirement; if you fit the calibrator on the training set, you overfit the calibrator to training noise. If you fit it on the test set, you inflate test performance and destroy your ability to measure the recalibrated model honestly.
- A **calibration function** `g: s ↦ p̂` that maps raw scores to probabilities. `g` is what you are about to fit.

The recalibrated pipeline is `p̂(x) = g(s(x))`. You then re-measure ECE, MCE, Brier, and the reliability diagram on the held-out test set — the same set of diagnostics from Chapter 2 — and report before-and-after.

Rule of thumb on set sizes: reserve ~10–20% of your labeled data as the calibration split, disjoint from train and test. If you already have a train / validation / test split, either carve a fresh calibration split (preferred) or use cross-validation with `sklearn.calibration.CalibratedClassifierCV` (see below). Using the validation set for both hyperparameter tuning *and* calibration is a mild leak but is common in practice; note it in the eval spec.

## Platt scaling

Platt (1999) proposed fitting a **one-dimensional logistic regression** on the raw model scores:

`p̂(x) = σ(a · s(x) + b),   where σ(z) = 1 / (1 + exp(−z))`.

The parameters `a` and `b` are fit by maximum likelihood on the calibration set. Two parameters, so the calibrator is very low-capacity.

**When Platt is appropriate.**

- The raw score is a linear-in-log-odds signal. This holds by construction for SVM margin scores (which is why Platt introduced it in the SVM context) and approximately for many boosted-tree scores.
- The miscalibration curve is **sigmoid-shaped** — the model's scores drift symmetrically towards the middle or the extremes, but the ordering is largely correct.
- You have a *small* calibration set (a few hundred to a few thousand items). Platt's two parameters are cheap to estimate and don't overfit.

**When Platt fails.**

- Non-monotone miscalibration — where the model is overconfident at the low end and underconfident at the high end simultaneously in a way a sigmoid cannot fix.
- Scores that are already probabilities from a well-calibrated model — Platt will *degrade* good calibration because you are fitting a sigmoid where the identity was closer.
- Multi-modal miscalibration where a single sigmoid cannot capture the shape.

**Practical note (Platt's own regularization).** Platt's original paper uses a Bayesian correction on the labels — replace `y ∈ {0, 1}` with `y_+ = (N_+ + 1) / (N_+ + 2)` and `y_− = 1 / (N_− + 2)` — to avoid overconfidence when the calibration set has extreme label proportions or when the sigmoid fit is against separable data. Most modern implementations skip this and just fit the logistic; `sklearn`'s `CalibratedClassifierCV(method="sigmoid")` does the plain-logistic version. It matters when the calibration set is small.

## Isotonic regression

Isotonic regression fits a **piecewise-constant, monotonically non-decreasing** function from `s(x)` to `p̂`. It is nonparametric — the fit takes as many "steps" as needed to minimize squared error subject to the monotonicity constraint. The pool-adjacent-violators algorithm (PAVA) computes it in `O(n log n)`.

**When isotonic is appropriate.**

- You have a **large calibration set** (thousands to tens of thousands of items). Isotonic's nonparametric flexibility is a liability at small `n` — it will overfit.
- The miscalibration curve has **complex, non-sigmoid shape** (steps, plateaus, sharp inflections). Isotonic handles them; Platt cannot.
- You only need the monotonicity property — the ordering of your model's scores is trusted and you just want the values to be right.

**When isotonic fails.**

- Small calibration sets. The staircase overfits. Rule of thumb: below ~1000 calibration samples, prefer Platt.
- The predicted probability collapses to a small number of distinct values (the piecewise-constant "steps"). This is a real behavior of isotonic and can cause problems downstream (e.g. threshold search sees only a handful of achievable probabilities).
- The scores fall outside the calibration set's range — isotonic clamps to the extrema, so a test-time score above the max training-time score is mapped to `1.0`. If you expect distribution shift at test time, this matters.

**Multi-class extension.** For a multi-class problem, isotonic (like Platt) is usually applied one-vs-rest and then normalized. `sklearn`'s `CalibratedClassifierCV(method="isotonic")` does this automatically.

## Temperature scaling

Temperature scaling (Guo et al. 2017) is the modern-neural-net specialty. Instead of learning a two-parameter sigmoid or a nonparametric monotone map on the *scores*, it learns a single scalar `T > 0` and divides the *logits* before the softmax:

`p̂_k(x) = softmax(z(x) / T)_k`.

`T > 1` softens the distribution (reduces overconfidence); `T < 1` sharpens it. `T` is fit by minimizing the negative log-likelihood on the calibration set.

**Why temperature scaling works so well for neural nets.**

- It preserves the argmax — recalibration cannot change the accuracy. This is important: some teams gate recalibration on "does it hurt accuracy," and temperature scaling automatically passes.
- It has one parameter. Overfitting the calibrator is effectively impossible even with a small calibration set.
- Deep classifiers trained with cross-entropy tend to be miscalibrated in a specific way: overconfident uniformly, with the sharpness of the softmax being the main problem. A single temperature captures that shape.

**When temperature scaling fails.**

- The miscalibration is not uniform — the model is overconfident on some classes and underconfident on others. A single `T` cannot fix that; you need per-class temperatures (vector scaling) or matrix scaling.
- Non-neural-net models where you do not have raw logits. Temperature scaling on already-probabilities is either undefined or equivalent to raising to a power; it's not the same procedure.
- Slice-specific miscalibration. If one slice is overconfident and another is underconfident, a single `T` splits the difference.

**Vector scaling** (per-class temperature) and **matrix scaling** (a linear map on the logits before softmax) generalize temperature scaling with more parameters. Guo et al. found that plain temperature scaling was competitive with or better than vector/matrix scaling across a wide range of image-classification models — the extra parameters overfit before they helped. That is model-family-dependent; for text classifiers and tabular models, vector scaling sometimes wins.

## Picking between the three

Not by feel. On data:

1. **Fit all three on the calibration set** — Platt (sigmoid), isotonic, temperature if the model has logits.
2. **Evaluate on the held-out test set** — ECE (debiased or adaptive-binned), MCE, Brier score, log loss.
3. **Pick the one with the best ECE / Brier at similar accuracy** — accuracy should be near-identical for Platt and temperature scaling; isotonic can shift accuracy slightly at the decision threshold because the mapping is nonparametric. If two methods tie, prefer the lower-capacity one (Platt over isotonic, temperature over Platt) for robustness to distribution shift.

Defaults if you are in a hurry:

- **Deep neural net, cross-entropy loss.** Temperature scaling. First try.
- **SVM, boosted tree, or shallow classifier with a small calibration set (< ~1000).** Platt.
- **Boosted tree with a large calibration set (~10000+), or a model with strange non-sigmoid miscalibration.** Isotonic.
- **Naive Bayes.** Both Platt and isotonic help materially. Naive Bayes is notorious for extreme overconfidence; recalibrate before you use the probabilities anywhere.

The Niculescu-Mizil & Caruana (2005) study is the empirical foundation of the "Platt for boosted trees and SVMs, isotonic for random forests with sufficient data" heuristic. Reproduce their comparison on your own model before you commit.

## Calibration under class imbalance and cost asymmetry

Recalibration on a calibration set with a different base rate than production is a version of the composition-shift problem from Chapter 1.

- If your calibration set was oversampled (positive rate boosted to ease training), Platt will produce a probability curve calibrated for the oversampled rate, not for production. The fix is one of: draw the calibration set with the production base rate; recalibrate on a small production-rate set; or reweight the calibration loss.
- If production has drifted since calibration, the ECE at prediction time is worse than the ECE you measured on the held-out set. Set a monitoring metric (Chapter 4) that catches this before the operating point degrades silently.

Cost-asymmetric problems (Chapter 4) do not usually change *how* you calibrate — calibration is about probabilities, cost enters at the decision — but they do change how much calibration error you can tolerate before it moves the operating point.

## Cross-validated calibration

If you cannot spare a fresh calibration split, `sklearn.calibration.CalibratedClassifierCV` fits the base classifier `k` times on `k−1` cross-validation folds and calibrates each on its held-out fold, then averages. This gives you calibrated probabilities from the same data as training without a separate hold-out.

Caveats:

- The `k` calibrators are averaged, which slightly smooths the calibrator.
- The base classifier's hyperparameters must be fixed before this process — you cannot use cross-validation for both hyperparameter selection and calibration in one pass without a nested split.
- Cross-validated calibration is the right default when you have limited data; a fresh, larger calibration split is the right default when you have plenty.

## Validating a recalibrator: what to actually report

For any recalibrated model, publish the following before-and-after table:

| Metric | Uncalibrated | Platt | Isotonic | Temperature |
|---|---|---|---|---|
| ECE (adaptive, 15 bins) | ... | ... | ... | ... |
| MCE | ... | ... | ... | ... |
| Brier score | ... | ... | ... | ... |
| Log loss | ... | ... | ... | ... |
| Accuracy at 0.5 | ... | ... | ... | ... |
| AUC | ... | ... | ... | ... |

Every cell with a bootstrap CI (mod-101 Chapter 4). AUC should be invariant across the methods (Platt, temperature) or nearly so (isotonic); if it changes materially with isotonic, you have overfit the calibrator. Accuracy should be invariant with temperature scaling by construction; a change signals a bug in the pipeline.

Also publish the reliability diagram for each candidate calibrator on the same axes so the shape improvement is visible.

## Concrete sketch

```python
import numpy as np
from sklearn.calibration import CalibratedClassifierCV
from sklearn.isotonic import IsotonicRegression
from sklearn.linear_model import LogisticRegression

# Assume: base_clf trained on X_train, y_train.
# Held-out calibration data (X_cal, y_cal), test data (X_test, y_test).
raw_cal = base_clf.decision_function(X_cal)      # or predict_proba(X_cal)[:, 1]
raw_test = base_clf.decision_function(X_test)

# Platt: one-dimensional logistic regression on the raw scores.
platt = LogisticRegression().fit(raw_cal.reshape(-1, 1), y_cal)
p_platt = platt.predict_proba(raw_test.reshape(-1, 1))[:, 1]

# Isotonic: piecewise-constant monotone fit.
iso = IsotonicRegression(out_of_bounds="clip").fit(raw_cal, y_cal)
p_iso = iso.predict(raw_test)

# Temperature scaling on logits (requires logits, e.g. torch model).
# T is fit by minimizing NLL on (logits_cal, y_cal).
# See Guo et al. 2017; a one-parameter LBFGS fit is standard.

# Then re-measure ECE / MCE / Brier from Chapter 2 on p_platt, p_iso, p_temp
# and pick the calibrator with the best composite score.
```

`sklearn.calibration.CalibratedClassifierCV(base_clf, method="sigmoid"|"isotonic", cv=5)` does the cross-validated version automatically and is what most codebases end up with.

## What can go wrong

- **Calibrating on the test set.** Fatal. Every downstream number is optimistic.
- **Calibrating on data with a different base rate than production.** The calibrator systematically biases probabilities. Match the base rate or reweight.
- **Reporting only the recalibrated ECE without the diagram.** ECE can be low with a distorted mapping that hurts a specific slice; the diagram catches it.
- **Choosing a method by convention rather than evidence.** "We always use temperature scaling because we're a deep learning shop" is a policy, not a decision. Run the table above.
- **Skipping the recalibration recheck after retraining.** Every retrain invalidates the calibrator. Refit on a fresh calibration split as part of the release process.
- **Assuming isotonic can rescue an ordering-broken model.** Isotonic preserves the input ordering; it cannot fix a model whose scores are non-monotone in the true probability. If the model is fundamentally miscalibrated because its *ordering* is wrong, no post-hoc calibrator helps — that is a modelling problem, not a calibration problem.

## Summary

Recalibration is a small, cheap, high-leverage step. Fit Platt, isotonic, and temperature scaling on a disjoint calibration set, evaluate on the held-out test set with the Chapter 2 diagnostics, and pick the calibrator with the best ECE and Brier at parity accuracy — preferring the lower-capacity method when it ties. Temperature scaling is the default for deep neural nets with cross-entropy loss, Platt is the default for SVMs and small calibration sets, isotonic is the default for models with non-sigmoid miscalibration and plenty of calibration data. Always publish the before-and-after table plus the reliability diagrams. Chapter 4 turns to the next question a well-calibrated model does not answer on its own: at what score threshold do we actually act?
