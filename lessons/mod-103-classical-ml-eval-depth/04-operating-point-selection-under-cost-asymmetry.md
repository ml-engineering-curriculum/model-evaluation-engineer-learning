# Operating Point Selection Under Cost Asymmetry

A classifier that outputs a score is not a decision. The decision comes from a **threshold** — the score above which the item is treated as positive. Pick the wrong threshold and you have a well-calibrated, well-discriminated model that ships the wrong outcome. Pick the right threshold without documenting how, and every retrain re-opens an argument that should have been decided once.

Almost every real classification problem is cost-asymmetric: false positives and false negatives do not cost the same. Fraud false-negatives cost the bank; fraud false-positives cost the customer. Medical-screening false-negatives cost lives; false-positives cost a follow-up visit. Spam false-positives cost trust; false-negatives cost inbox clutter. Getting the threshold right is where the eval hands the product a shipping-quality artifact.

This chapter connects the PR / ROC curve to a cost matrix, shows how to pick a threshold under three common framings (expected utility, Neyman–Pearson, and a fixed operating budget), and lays out how to report the chosen operating point defensibly.

## The confusion matrix and its four costs

For a binary classifier with a threshold `t`, define the four outcomes:

- **TP** — true positive; classifier says positive, truth is positive.
- **FP** — false positive; classifier says positive, truth is negative.
- **TN** — true negative; classifier says negative, truth is negative.
- **FN** — false negative; classifier says negative, truth is positive.

Each has an associated cost `c_TP, c_FP, c_TN, c_FN` — the business consequence of that outcome. Typically `c_TP` and `c_TN` are near zero (or negative, i.e. utility) and the costs live on `c_FP` and `c_FN`. What matters is the **ratio** `c_FN / c_FP`, since a positive multiplicative constant on all four does not change the argmin.

The output is the **expected cost per prediction** at threshold `t`:

`E[cost | t] = π · (c_TP · TPR(t) + c_FN · FNR(t)) + (1 − π) · (c_FP · FPR(t) + c_TN · TNR(t))`,

where `π = P(Y = 1)` is the class prior, `TPR = TP / (TP + FN)` is the true-positive rate (recall), `FPR = FP / (FP + TN)` is the false-positive rate, and `FNR = 1 − TPR`, `TNR = 1 − FPR`.

Threshold selection is: find `t*` that minimizes `E[cost | t]`.

For a well-calibrated classifier (Chapters 2–3), the **Bayes-optimal threshold** has a closed form:

`t* = (c_FP − c_TN) / ((c_FN − c_TP) + (c_FP − c_TN))`.

With `c_TP = c_TN = 0`: `t* = c_FP / (c_FP + c_FN)`. If false negatives cost 4× false positives, `t* = 1 / (1 + 4) = 0.2`. The classifier should flag anything scored above `0.2`.

This closed form is the reason calibration matters. If the scores are not probabilities, `t*` computed this way is meaningless.

## PR and ROC curves: what they show and what they hide

The **ROC curve** plots TPR against FPR as `t` varies from `1` to `0`. AUC is the area under it and equals `P(score(positive) > score(negative))` — a threshold-free discrimination metric.

The **PR curve** plots precision against recall as `t` varies. Precision is `TP / (TP + FP)`, recall is `TPR`. Area under PR is a discrimination metric too, but it is base-rate-dependent: at a `1%` prevalence, precision is dominated by the FP-vs-TP ratio, which drops fast as recall rises.

**Rule of thumb.** Use the ROC curve when the positive class is not extremely rare (say, prevalence > 5%). Use the PR curve when it is. ROC on a `1%`-prevalence problem produces optimistic-looking curves: FPR of `1%` looks great, but if 99% of the ground-truth negatives are in play, `1%` FPR is still `~1000 FPs per 100,000 items` — which the precision axis makes visible and the FPR axis buries. Precision-recall is base-rate-aware; ROC is not.

**What both hide.**

- The *threshold* on the curve. A single (precision, recall) point on the PR curve is the wrong artifact to ship. Ship the threshold, the confusion matrix at that threshold, and the CIs on both axes.
- The *cost* at each point. The curve is agnostic to cost; the operating point is not.
- The *sample size in each region*. A PR curve at recall `0.99` is often estimated on ~2% of the labeled data (the highest-scored items). CIs on precision at that recall are wide, and the curve does not show it.

Always plot the curve with:

- The chosen operating point marked and labelled with `(threshold, precision, recall)`.
- A point-wise CI band from the bootstrap (mod-101 Chapter 4).
- The prevalence in the caption.

## Three framings for picking the operating point

Which framing you use depends on what the business gave you.

### 1. Expected-utility maximization

You are given `(c_TP, c_FP, c_TN, c_FN)`. Compute the empirical expected cost on the held-out set at each candidate threshold and pick the minimizer.

```python
import numpy as np
def expected_cost(y_true, y_score, thresholds, c_tp, c_fp, c_tn, c_fn):
    y_true = np.asarray(y_true).astype(bool)
    y_score = np.asarray(y_score)
    n = len(y_true)
    costs = []
    for t in thresholds:
        y_pred = y_score >= t
        tp = int(((y_pred) & y_true).sum())
        fp = int(((y_pred) & ~y_true).sum())
        tn = int(((~y_pred) & ~y_true).sum())
        fn = int(((~y_pred) & y_true).sum())
        costs.append(c_tp*tp + c_fp*fp + c_tn*tn + c_fn*fn)
    return np.array(costs) / n
```

Grid over the sorted unique scores (there are at most `n` distinct thresholds) or over a candidate list. Take `argmin`. Report the chosen threshold, the expected cost at it, and the bootstrap CI on the expected cost.

The most common mistake: sourcing `c_FP` and `c_FN` from the analyst's intuition rather than from a product / finance / trust-and-safety conversation. The costs should be numbers the business owns and reviews, not numbers the model owner invented to make the operating point feel right.

### 2. Neyman–Pearson: constrained optimization

Sometimes the business does not give you a cost ratio; they give you a **constraint**. Two typical forms:

- "FPR must be at most 5%" — the ratio of false positives to true negatives must not exceed 5%. Common in security / spam / content-moderation settings where a hard cap on user-visible error rate matters.
- "Precision must be at least 90%" — the ratio of true positives to flagged items must be at least 90%. Common in review-queue settings where the reviewer capacity is fixed.

Under the constraint, maximize the other axis. Pick the threshold that gives the highest TPR subject to `FPR ≤ α`, or the highest recall subject to `precision ≥ β`. Report the chosen threshold, both axes' values at that threshold, and a CI on the achievable operating point.

This is the Neyman–Pearson formulation from classical hypothesis testing. It is often more honest than expected-utility maximization when the business genuinely does not know the cost ratio but does know the constraint.

### 3. Fixed operating budget

Sometimes the constraint is on *volume*, not rate. "We can review 500 items per day" or "we can send at most 10,000 alerts this week." Pick the threshold that yields exactly the budgeted number of positive predictions. Under a fixed budget, higher-scored items are chosen first — which is the same as picking the `k` highest scores where `k` is your budget.

Report the threshold, the expected precision at that budget (i.e. the fraction of the `k` selected items that will be true positives), and how the budget-vs-precision curve looks in the neighborhood so the budget owner can trade a marginal item for a marginal precision improvement.

## The 0.5 default is almost never right

The scikit-learn convention `y_pred = model.predict(X)` uses threshold `0.5`. `0.5` is Bayes-optimal only when `c_FP = c_FN` and `π = 0.5`. Almost no production classifier has both properties. If the product review has not chosen a threshold, the default is telling them "assume equal costs and balanced prevalence," which is a very strong claim to make silently.

Never ship a classifier at `0.5` because it was the default. Either the threshold is explicitly `0.5` because you did the analysis and it was optimal, or the threshold is something else and the analysis picked it, or you have not done the analysis and the classifier is not shippable yet.

## Reporting the operating point defensibly

For the eval report, the operating point section should contain:

1. **The framing.** Expected-utility, Neyman–Pearson, or fixed-budget. If expected-utility, the cost matrix and where it came from. If Neyman–Pearson, the constraint and its source. If fixed-budget, the budget and its owner.
2. **The chosen threshold** `t*` with 3 significant figures.
3. **The confusion matrix at `t*`** (TP, FP, TN, FN counts), with bootstrap CIs (mod-101 Chapter 4) around at least precision, recall, FPR, and any composite metric the business cares about.
4. **The expected cost or constrained metric at `t*`** with its CI.
5. **A sensitivity analysis.** Small perturbations of the cost ratio (or the constraint) and how much `t*` moves. If a `10%` change in `c_FN / c_FP` moves the threshold by an order of magnitude, the operating point is fragile and the business should know.
6. **The PR and/or ROC curve** with `t*` marked and a CI band.
7. **Per-slice operating points.** For the slicing axes from Chapter 1, either use the global `t*` and report per-slice precision / recall, or (with a fairness caveat, Chapter 5) allow the threshold to vary per slice. The latter is a fairness decision and needs to be argued.
8. **The recalibration status** (Chapter 3). If the operating point was chosen on uncalibrated scores, the Bayes-optimal closed-form does not apply; disclose the choice.

## Per-slice thresholds and the fairness question

The Bayes-optimal threshold depends on `π` (the prevalence) and on the costs. If prevalence varies across slices, the Bayes-optimal threshold also varies. This is where operating-point selection collides with fairness (Chapters 5–6).

- **Single global threshold, same for all slices.** Simplest, easiest to justify in policy, but produces different per-slice precision/recall trade-offs. In the extreme, one slice bears most of the false-positive rate.
- **Per-slice thresholds tuned to a per-slice constraint** (e.g. equal FPR across slices). Achieves the fairness constraint by construction but requires you to explicitly justify treating slices differently — which is a legal / policy question in many jurisdictions.
- **Threshold randomization** (Hardt et al. 2016). Occasionally used to achieve exact equalized odds; requires stochastic decisions, which are often unacceptable in high-stakes settings.

Do not silently ship per-slice thresholds without the fairness discussion. Chapter 5 is the framing.

## Bootstrap CIs on the operating point

The threshold `t*` is estimated from finite data. It is itself a random variable. Report a CI around it.

- **Percentile bootstrap on `t*`.** Resample the labeled set with replacement `B` times, recompute the operating-point-selection procedure on each resample, collect the `B` `t*` values, take the 2.5th and 97.5th percentiles. If the interval is wide (say, more than `0.05` on a `[0, 1]` score), the threshold is not well-identified by the data and you need either more data or a more constrained framing (Neyman–Pearson tends to be more stable).
- **CI on the operating-point metrics.** Percentile bootstrap on precision, recall, FPR, and expected cost at the chosen `t*`. These are what the report ships. Wide intervals here mean the operating point cannot be defended against small data changes; consider raising the sample size or reporting a range of acceptable thresholds instead of a single point.

## Common failure modes

- **`0.5` as the default with no discussion.** Already covered. This is the most common failure.
- **Optimizing an accuracy or F1 score to pick the threshold.** F1 is `2·P·R / (P + R)` — a specific cost trade-off in disguise. Optimizing F1 to pick the operating point is the same as saying `c_FP = c_FN` weighted by class prior; check whether that is actually the business's intent before defaulting to F1.
- **Optimizing AUC.** AUC does not have a threshold. If someone reports "we picked the operating point that maximized AUC," they picked no threshold at all — AUC is invariant to `t`.
- **Optimizing on the test set.** The operating point is a *hyperparameter*. Choose it on a hold-out set that is separate from the reporting test set. Otherwise the reported precision / recall at the operating point are optimistic.
- **Reporting a single (precision, recall) pair without the threshold and confusion matrix.** Deferred to the reader to reconstruct; often they cannot.
- **Sensitive attributes silently in the cost function.** If `c_FN` is different for one demographic group than another and this shows up in the operating-point choice, that is a fairness decision (Chapter 5). Document it explicitly rather than embedding it in the cost weights.
- **Assuming the cost ratio is stable over time.** Business costs drift (a new SLA, a regulatory change, a competitor's move). Recompute the operating point when the costs change, not just when the model changes.

## Concrete sketch

Combining the elements from Chapters 2, 3, and this one:

```python
import numpy as np
from sklearn.metrics import precision_recall_curve, roc_curve

def choose_threshold_expected_cost(y_true, y_score, c_fp, c_fn,
                                   c_tp=0.0, c_tn=0.0):
    # Enumerate the ~n candidate thresholds implicit in the sorted scores.
    order = np.argsort(-y_score)
    y_sorted = np.asarray(y_true)[order]
    scores_sorted = np.asarray(y_score)[order]
    # Vectorized cumulative-count trick:
    tp_cum = np.cumsum(y_sorted == 1)
    fp_cum = np.cumsum(y_sorted == 0)
    P = int((y_sorted == 1).sum()); N = int((y_sorted == 0).sum())
    fn_cum = P - tp_cum; tn_cum = N - fp_cum
    costs = (c_tp*tp_cum + c_fp*fp_cum + c_tn*tn_cum + c_fn*fn_cum) / len(y_sorted)
    best = int(np.argmin(costs))
    return {
        "threshold": float(scores_sorted[best]),
        "tp": int(tp_cum[best]), "fp": int(fp_cum[best]),
        "tn": int(tn_cum[best]), "fn": int(fn_cum[best]),
        "expected_cost": float(costs[best]),
    }

def choose_threshold_neyman_pearson_fpr(y_true, y_score, fpr_max):
    fpr, tpr, thr = roc_curve(y_true, y_score)
    ok = fpr <= fpr_max
    if not ok.any():
        raise ValueError(f"No threshold achieves FPR <= {fpr_max}")
    best = int(np.argmax(tpr[ok]))
    idx = np.flatnonzero(ok)[best]
    return {"threshold": float(thr[idx]), "fpr": float(fpr[idx]),
            "tpr": float(tpr[idx])}

def choose_threshold_min_precision(y_true, y_score, precision_min):
    precision, recall, thr = precision_recall_curve(y_true, y_score)
    # precision_recall_curve returns thresholds of length len(precision)-1.
    ok = precision[:-1] >= precision_min
    if not ok.any():
        raise ValueError(f"No threshold achieves precision >= {precision_min}")
    idx = int(np.argmax(recall[:-1][ok]))
    idx = np.flatnonzero(ok)[idx]
    return {"threshold": float(thr[idx]), "precision": float(precision[idx]),
            "recall": float(recall[idx])}
```

Wrap each in a bootstrap (mod-101 Chapter 4) to get a CI on the chosen threshold and on the achieved operating point.

## Summary

An operating point is a decision — a threshold, chosen against an explicit cost framing, defended with a CI, and reported with the confusion matrix at that threshold. Three framings cover almost every case: expected utility with a cost matrix, Neyman–Pearson with a constraint, and a fixed operating budget with a `k`-highest rule. The Bayes-optimal threshold has a closed form for well-calibrated classifiers, which is why Chapters 2 and 3 come first; without calibration, the closed form does not apply. Never ship `0.5` because it is the default. Report the chosen threshold, the confusion matrix at that threshold, bootstrap CIs, a sensitivity analysis on the cost ratio, and per-slice operating points if slices differ meaningfully — and be prepared to defend per-slice thresholds as a fairness decision, which Chapters 5–6 unpack.
