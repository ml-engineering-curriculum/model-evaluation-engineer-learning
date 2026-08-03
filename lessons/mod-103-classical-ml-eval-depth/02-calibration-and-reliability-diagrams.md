# Calibration and Reliability Diagrams

A classifier that outputs `0.9` is making a claim: of every hundred items I score at `0.9`, roughly ninety are positive. Calibration is the discipline of checking whether that claim is true. It is separate from accuracy, separate from discrimination (AUC), and separate from the operating point (Chapter 4). A model can be highly accurate and badly miscalibrated; a model can have a strong AUC and still be systematically over- or under-confident. Downstream systems that consume scores — cost-weighted decisions, threshold selection, human-in-the-loop routing, uncertainty-aware retrieval — depend on the scores being interpretable as probabilities.

This chapter defines calibration, shows how to measure it with reliability diagrams and Expected Calibration Error (ECE), and lays out the traps in the standard reports. Chapter 3 covers what to do when the answer is "badly miscalibrated."

## What calibration is (and is not)

A binary classifier that outputs a score `p̂(x) ∈ [0, 1]` is **perfectly calibrated** if, for every value `p ∈ [0, 1]`,

`P(Y = 1 | p̂(X) = p) = p`.

Read aloud: conditional on the model saying `0.7`, the empirical probability of the positive class is `0.7`. This is a claim about the model's scores across the whole test distribution.

**What calibration is not.**

- It is not accuracy. A model that outputs `0.5` for every input is perfectly calibrated iff the base rate is `0.5`, but has zero discrimination.
- It is not AUC. AUC is invariant to any monotonic rescaling of the scores; you can destroy calibration completely without moving AUC by one bit.
- It is not "high-confidence predictions are more likely correct." That is a weaker property (sometimes called *resolution*); calibration requires the specific numeric interpretation.

The two questions to hold separately are discrimination — "does the model order items correctly?" (AUC/PR) — and calibration — "do the scores mean what they say?" (this chapter). Both matter; neither implies the other.

## Why classical ML models are often miscalibrated by default

- **Logistic regression** with maximum likelihood on a well-specified model tends to be well-calibrated *on the training distribution*. Out of distribution it drifts.
- **Naive Bayes** produces scores that are typically overconfident because the independence assumption compounds evidence. Predictions cluster near `0` and `1`.
- **SVMs** produce margin scores, not probabilities; sigmoid or isotonic post-processing is required to get probabilities at all.
- **Random forests** are typically miscalibrated with sigmoid-shaped miscalibration — pushing scores toward the middle — because averaging trees blunts the tails.
- **Gradient-boosted trees** with log-loss are usually well-calibrated in the middle of the score range and slightly under-confident at the extremes.
- **Deep neural networks** trained with cross-entropy are typically overconfident on modern architectures — the "modern neural nets are miscalibrated" finding from Guo et al. 2017. The result generalizes to most large classifiers trained past the point of near-zero training loss.

Empirical calibration on your specific data is required regardless of the model class. Priors and folklore are not substitutes for the reliability diagram.

## The reliability diagram

The reliability diagram is the graphical summary of calibration.

1. Bin the predictions by score into `M` bins — e.g. 10 fixed-width bins on `[0, 1]`.
2. For each bin `m`, compute the average predicted score `p̄_m` and the empirical positive rate `ȳ_m`.
3. Plot `ȳ_m` on the y-axis against `p̄_m` on the x-axis, one point per bin, ideally with the bin count as marker size or on a secondary axis. Draw the identity line `y = x`.

Interpretation:

- Points on the identity line: calibrated in that bin.
- Points below the line: overconfident (model says `p̄ = 0.9`, empirical is `0.7`).
- Points above the line: underconfident (model says `p̄ = 0.3`, empirical is `0.5`).
- Bin count matters. A single bin with `n_m = 4` will look wildly off the diagonal from sampling noise alone.

Always plot the histogram of scores as a companion. A reliability diagram with beautiful bin averages that all live in `[0, 0.05]` and `[0.95, 1.0]` (typical of overconfident deep nets) is a very different story than one whose scores are spread across `[0, 1]`. The score histogram tells the reader where the density is; the reliability plot tells them how honest the scores are there.

## Expected Calibration Error (ECE) and Maximum Calibration Error (MCE)

The standard scalar summary is Expected Calibration Error:

`ECE = Σ_m (n_m / n) · |ȳ_m − p̄_m|`

— the weighted average absolute gap between the empirical rate and the mean predicted score, weighted by bin count. Lower is better; 0 means perfectly calibrated (up to the bin resolution).

Maximum Calibration Error takes the worst bin:

`MCE = max_m |ȳ_m − p̄_m|`.

Report both. ECE tells you "on average how far off am I;" MCE tells you "what is the worst bin, which is where the safety-relevant decisions live."

## Fixed-width vs. adaptive (equal-mass) binning

Fixed-width binning (bins on `[0, 0.1), [0.1, 0.2), …`) is the default because it is intuitive. It has two problems.

**Bin-density imbalance.** With an overconfident model, 90% of scores can end up in the `[0.9, 1.0]` bin. The other nine bins get very few samples, their per-bin averages have huge variance, and `ECE` is dominated by whichever thin bins happened to draw. Report ECE, and it looks like the model is well-calibrated because the largest bin (the one that dominates the weighted average) is easy to hit.

**Sensitivity to bin count.** ECE with 5 bins and ECE with 100 bins can differ by an order of magnitude. There is no single correct choice, but you must fix and document it.

The adaptive-binning fix (sometimes called quantile binning or equal-mass binning) puts an equal number of samples in each bin: sort the scores, split into `M` groups of `n/M` each, and compute the bin edges from the data. Every bin has meaningful sample size; the plot no longer over-weights sparsely-populated bins. `ECE` computed on adaptive bins is more stable across bin counts.

The trade-off: adaptive bins have irregular width, so the diagram is harder to read directly. Report both if you have room; if you have to pick one, adaptive is safer for scalar comparison.

Alternatives you will see in the literature and libraries:

- **`sklearn.calibration.calibration_curve`** — supports both `strategy="uniform"` (fixed width) and `strategy="quantile"` (adaptive).
- **Debiased ECE / Kernel ECE (Kumar et al. 2019).** Corrects the finite-sample bias in ECE and reduces its dependence on the bin count. Preferable when you need a defensible scalar to gate a shipping decision on.
- **Adaptive Calibration Error (Nixon et al. 2019).** A refinement of adaptive binning with weighted contributions.

## Brier score and its decomposition

Brier score is the mean squared error of the probability predictions:

`BS = (1/n) · Σ_i (p̂_i − y_i)²`.

It is a **proper scoring rule**: it is minimized in expectation only when the reported probability equals the true conditional probability. That makes it appealing as a single-number diagnostic that penalises miscalibration and lack of resolution jointly.

Brier score decomposes (Murphy 1973) as

`BS = Reliability − Resolution + Uncertainty`,

with the three terms:

- **Reliability** — the calibration term. Zero when the model is perfectly calibrated. This is close to ECE² in the bin-averaged form.
- **Resolution** — how much the conditional positive rates in each bin differ from the overall base rate. Higher is better.
- **Uncertainty** — the variance of the label; a property of the data, not the model. `p̄(1 − p̄)` where `p̄` is the base rate.

The decomposition is why Brier is the right single-number summary when you must have one: it separately penalises "you have wrong probabilities" (reliability) and rewards "you distinguish easy from hard items" (resolution).

Log loss (cross-entropy) is also a proper scoring rule and shares this structure. It is much more sensitive than Brier to confident wrong predictions (the `-log(0)` blow-up) and correspondingly less robust to outliers; pick based on whether you want a bounded-influence diagnostic (Brier) or a decision-theoretically-clean loss for training (log loss).

## Calibration under class imbalance and slicing

At extreme class imbalance, the score distribution collapses toward the base rate and calibration diagnostics get harder.

- **Base-rate adjustment.** For an eval set drawn with a base rate different from production, calibration you measure on the eval set is not the calibration production sees. Reweight the bins by the production base rate or, more cleanly, evaluate calibration on a sample drawn with the production base rate.
- **Per-slice calibration.** Global calibration can be excellent while a specific slice is badly miscalibrated. If Chapter 1 gave you slicing axes, compute calibration per slice, report per-slice ECE, and flag slices with ECE more than 2× the aggregate. This is often where a slice-specific recalibration (Chapter 3) is warranted.
- **Multi-class calibration.** For `K` classes, the standard is *top-label calibration*: for each item, take the model's predicted class and its associated probability; do the reliability analysis on those. Class-wise calibration (calibration of each class's marginal probability) is stricter and often more useful when downstream decisions use the full probability vector.

## What a good calibration report contains

1. **The reliability diagram** with both fixed-width and adaptive bins, and the score histogram overlay.
2. **ECE, MCE, and Brier score** with 95% bootstrap CIs (mod-101 Chapter 4).
3. **Debiased or kernel ECE** if the shipping decision hinges on the calibration number.
4. **Per-slice calibration** for the axes from Chapter 1, at minimum ECE per slice.
5. **Base-rate context** — the eval-set positive rate and the production positive rate (if known), so a reader can spot base-rate drift as a calibration issue rather than a modelling issue.
6. **Recalibration recommendation** — Chapter 3 covers the choice, but the calibration report should end with "recalibrate with X" or "no recalibration needed" and the evidence for that call.

## Concrete sketch: measuring calibration in scikit-learn

```python
import numpy as np
from sklearn.calibration import calibration_curve
from sklearn.metrics import brier_score_loss

def ece(y_true, y_score, n_bins=15, strategy="quantile"):
    prob_true, prob_pred = calibration_curve(
        y_true, y_score, n_bins=n_bins, strategy=strategy,
    )
    # calibration_curve drops empty bins, so recompute weights on the retained bins.
    bin_edges = np.quantile(y_score, np.linspace(0, 1, n_bins + 1)) \
        if strategy == "quantile" else np.linspace(0, 1, n_bins + 1)
    counts, _ = np.histogram(y_score, bins=bin_edges)
    used_counts = counts[counts > 0]
    weights = used_counts / used_counts.sum()
    return float(np.sum(weights * np.abs(prob_true - prob_pred)))

def mce(y_true, y_score, n_bins=15, strategy="quantile"):
    prob_true, prob_pred = calibration_curve(
        y_true, y_score, n_bins=n_bins, strategy=strategy,
    )
    return float(np.max(np.abs(prob_true - prob_pred)))

# Bootstrap CIs (mod-101 ch. 4) around ECE/Brier so the number isn't reported bare.
def bootstrap_ece(y_true, y_score, n_boot=1000, seed=0, **kw):
    rng = np.random.default_rng(seed)
    n = len(y_true)
    idx = rng.integers(0, n, size=(n_boot, n))
    return np.array([ece(y_true[i], y_score[i], **kw) for i in idx])

# Brier is easy:
brier = brier_score_loss(y_true, y_score)
```

For the reliability diagram itself, `sklearn.calibration.CalibrationDisplay.from_predictions` renders it directly. For the histogram overlay and per-slice plots, drop into `matplotlib` / `seaborn`.

## Common failure modes in calibration reports

- **Reporting ECE on 10 fixed bins without the histogram.** The number looks fine; the diagram would show 92% of scores in the last bin. Always publish the histogram.
- **Reporting ECE with no CI.** With `n ≤ 5000`, ECE has a bootstrap CI wider than half the score range. Ship without a CI and you are asking a reader to interpret a point estimate that is dominated by sampling noise.
- **Averaging away a slice-specific miscalibration.** A 3% ECE globally can hide a 25% ECE on a minority slice. Per-slice calibration is not optional in a fairness- or safety-relevant setting.
- **Confusing calibration with accuracy.** "Our model is 92% accurate" is not a calibration claim. If the report cites accuracy as evidence for the probability interpretation, it is wrong.
- **Retraining without recalibrating.** Any change to the training distribution, the loss function, the temperature (softmax scaling), or the feature pipeline can invalidate the calibration curve. Recalibration is cheap; skipping the check after a retrain is how ship-time calibration silently rots.

## Summary

Calibration asks whether a score of `p` corresponds to an empirical positive rate of `p`. Measure it with a reliability diagram (with the score histogram alongside it) and summarize with ECE, MCE, and Brier score, all with bootstrap CIs. Prefer adaptive (quantile) binning to fixed-width for ECE, and report the bin count explicitly. Recognize that classical ML models are usually miscalibrated by default in shape-specific ways (Naive Bayes overconfident, random forests sigmoid-miscalibrated, modern deep nets overconfident at the extremes), so the calibration check is required, not optional. Report calibration per slice as well as globally, and end the report with a recalibration recommendation. Chapter 3 covers how to act on that recommendation: Platt scaling, isotonic regression, and temperature scaling, and how to pick between them.
