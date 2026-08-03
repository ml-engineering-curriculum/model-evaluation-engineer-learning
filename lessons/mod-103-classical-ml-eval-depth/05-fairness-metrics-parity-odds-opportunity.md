# Fairness Metrics: Demographic Parity, Equalized Odds, Equal Opportunity

Fairness measurement is a specific application of per-slice metrics (Chapter 1) in which the slicing axis is a **legally- or ethically-protected attribute** — race, sex, age, disability, national origin, religion, and often a broader set your organization has committed to (income, geography, marital status, etc.). What changes from Chapter 1 is not the arithmetic; it is what a gap between slices *means*, and how the operating point (Chapter 4) interacts with the choice.

This chapter defines the standard fairness metrics precisely, shows how they relate to each other, and lays out how to measure them without pretending the measurement alone answers the policy question. Chapter 6 covers the impossibility results — the reason you cannot satisfy all the definitions at once — and the tooling.

## Setup and notation

Let `A` be the **sensitive attribute** (e.g. `race ∈ {A₁, A₂, ...}` or a binary `A ∈ {0, 1}`), `Y` the true label, `Ŷ` the model's prediction (typically a hard-classified `0` or `1` from Chapter 4's operating point), and `p̂ = P(Y = 1 | X)` the model's score.

The five most-cited fairness definitions all compare the model's behavior on subpopulations defined by `A`. They differ in *which* conditional probability is being equalized. Each has a name (sometimes several), a precise formula, and a set of behaviors it does and does not guarantee.

## Demographic parity (statistical parity)

`P(Ŷ = 1 | A = a) = P(Ŷ = 1 | A = a')` for all pairs of protected values `(a, a')`.

The positive-prediction rate is the same across groups. Also called **statistical parity** or **group parity**.

**Common measurement forms.**

- **Demographic parity difference**: `P(Ŷ = 1 | A = a) − P(Ŷ = 1 | A = a')`. Report the maximum over pairs.
- **Disparate impact ratio**: `P(Ŷ = 1 | A = a) / P(Ŷ = 1 | A = a')`. Report the minimum over pairs.
- **The four-fifths rule.** In US employment law (EEOC Uniform Guidelines on Employee Selection Procedures, 29 CFR §1607.4(D)), a selection rate for any protected group less than four-fifths (80%) of the highest group's selection rate is generally regarded as evidence of adverse impact. This is a *guideline*, not a bright line — an 82% ratio is not "safe" and an 78% ratio is not automatically "discriminatory," but the number is widely cited as a screening threshold. Know it because it will come up.

**What demographic parity does not require.**

- It does not require the model to be accurate.
- It does not require the model to be calibrated.
- It does not require the model's errors to fall evenly across groups.

**When demographic parity is appropriate.**

- The base rates `P(Y = 1 | A)` are believed to be the same across groups (or the difference in base rates is itself due to historical inequity you do not want the model to encode). Common in lending, hiring, and school admissions arguments.
- The action following `Ŷ = 1` is a *resource allocation* (a loan, an interview, an ad impression) and equal access across groups is the policy goal.

**When demographic parity is not appropriate.**

- The base rates genuinely differ for reasons the model is entitled to use (e.g. probability of a medical condition genuinely differs by biological sex in some settings). Forcing equal selection rates on unequal base rates injects error to satisfy the parity.
- The action following `Ŷ = 1` is a *quality-of-service* prediction (e.g. content classification, fraud detection) and you actually want the model to reflect the true rate.

## Equalized odds

`P(Ŷ = 1 | A = a, Y = 1) = P(Ŷ = 1 | A = a', Y = 1)` **and** `P(Ŷ = 1 | A = a, Y = 0) = P(Ŷ = 1 | A = a', Y = 0)`.

Equal TPR (true positive rate, i.e. recall) **and** equal FPR (false positive rate) across groups. Hardt, Price, and Srebro introduced this in 2016 as a response to the observed unfairness of demographic parity when base rates differ legitimately.

**Common measurement forms.**

- **True positive rate difference**: `|TPR_a − TPR_a'|`.
- **False positive rate difference**: `|FPR_a − FPR_a'|`.
- **Equalized odds difference**: `max(|TPR_a − TPR_a'|, |FPR_a − FPR_a'|)` — a single-number summary.

**What equalized odds requires.**

- If you are a true positive, your probability of being correctly identified is the same regardless of group.
- If you are a true negative, your probability of being incorrectly flagged is the same regardless of group.

**Where equalized odds fails.**

- When the two rates cannot be jointly equalized without changing the model or the threshold per group. The Hardt et al. paper proposes threshold randomization to achieve exact equalized odds; stochastic decisions are often unacceptable in high-stakes settings.
- When the group-conditional PR / ROC curves differ substantially (a model can be more discriminating on one group than another). No threshold choice can equalize both rates simultaneously if the ROC curves are shaped differently.

## Equal opportunity

`P(Ŷ = 1 | A = a, Y = 1) = P(Ŷ = 1 | A = a', Y = 1)`.

Equal TPR across groups. A weaker version of equalized odds that only equalizes the true-positive rate, not the false-positive rate.

**When equal opportunity is appropriate.**

- The "opportunity" framing: the harm of missing a true positive (a qualified applicant not hired, a needy patient not treated) is what you are trying to equalize. FPR is a cost you are willing to let vary across groups.

Equal opportunity is often the fairness metric of choice in resource-allocation settings where the positive class is the beneficial outcome and the FPR falls on the institution rather than the individual.

## Predictive parity (positive predictive value parity)

`P(Y = 1 | Ŷ = 1, A = a) = P(Y = 1 | Ŷ = 1, A = a')`.

Equal precision across groups: given that the model says positive, the empirical positive rate is the same regardless of group.

The COMPAS controversy (Angwin et al. 2016 vs. Dieterich et al. 2016) turned on predictive parity vs. equalized odds. COMPAS satisfied predictive parity across race but failed equalized odds — the FPR was higher for Black defendants. Both sides were arithmetically correct. What differed was which definition they were prioritizing. Chapter 6 covers why you cannot generally have both.

## Calibration within groups (calibration parity)

`P(Y = 1 | p̂ = p, A = a) = p` for all `p` and `a`.

The model is well-calibrated (Chapter 2) *conditional on each group*. A stronger property than global calibration.

Calibration within groups is often the "keep the probabilities honest" framing. If a downstream system consumes the score as a probability, it should mean the same thing across groups; otherwise the downstream decision is silently biased.

## The three-by-three summary

The five metrics above can be organized by what is fixed and what is compared:

| Definition | Fixes... | Compares across groups |
|---|---|---|
| Demographic parity | nothing | `P(Ŷ = 1)` |
| Equalized odds | `Y` (both values) | `P(Ŷ = 1 | Y = y)` for each `y` |
| Equal opportunity | `Y = 1` | `P(Ŷ = 1 | Y = 1)` |
| Predictive parity | `Ŷ = 1` | `P(Y = 1 | Ŷ = 1)` |
| Calibration within groups | `p̂` | `P(Y = 1 | p̂)` |

Each answers a different question. The impossibility results in Chapter 6 are essentially "you can only fix one of these things at a time when base rates differ across groups."

## Choosing the sensitive attribute

The metric arithmetic is easy. What breaks in real projects is what `A` is.

**Data availability.** Many organizations do not collect the sensitive attributes directly (US employment law, for example, often forbids soliciting them). Common approaches:

- **Self-reported at signup.** Best when available and consented.
- **Bayesian Improved Surname Geocoding (BISG).** Estimates race and ethnicity from surname and geocoded address. Standard in US financial-services fairness auditing; has documented bias (over-predicts some groups, under-predicts others) that must be characterized before it is used.
- **Not measured at all.** Some organizations argue "we do not collect protected attributes, so we cannot bias against them." This is provably wrong in general (see Dwork et al. 2012 "Fairness Through Awareness"): models routinely learn proxies for protected attributes from other features. The absence of measurement is the absence of measurement, not the absence of disparity.

**Intersectional fairness.** Single-attribute fairness (women vs. men, Black vs. white) can hide within-intersection disparities. Buolamwini & Gebru (2018) showed that commercial face-classification systems performed worst on darker-skinned women — an intersection that was invisible in the marginal (race-only or gender-only) analyses. The measurement fix is to slice on the intersection when the sample size supports it; the practical fix is to plan for sparse intersections (some cells will have `n < 30`) and to publish sparsity flags rather than silently pooling.

**Categorical vs. continuous attributes.** Age, income, and geography are often continuous or ordinal. Bin them explicitly (as in Chapter 1) and document the boundaries. Never bin continuous attributes silently or in an axis-optimized way; the choice of bin boundaries substantially changes the reported metrics.

## Reporting a fairness measurement

The output is not one number. It is a report that contains, at minimum:

1. **The sensitive attribute definition.** What `A` is, how it was collected or inferred, its known measurement error (for inferred attributes), and the values considered.
2. **The reference or comparison group.** Explicitly state which group is the reference and why (regulatory requirement, largest group, or a specific policy choice). All differences and ratios are relative to that group.
3. **The metric(s) chosen** and why. Not one — usually three or four, because each answers a different question. At minimum: demographic parity, TPR and FPR by group (equalized odds decomposition), and calibration within groups. Include predictive parity if the downstream decision depends on precision.
4. **Point estimates and CIs.** Every metric with a Wilson or bootstrap CI (mod-101 Chapters 3–4). Fairness disputes routinely hinge on whether a difference is real or sampling noise; without CIs you cannot answer that.
5. **The operating point.** The threshold in effect (Chapter 4). Fairness numbers move with the threshold, so a fairness report at threshold `0.5` and a fairness report at threshold `0.3` are different reports.
6. **Intersectional slices.** At least the two-way intersections of the most-important protected attributes, subject to the sparsity flag from Chapter 1.
7. **Interpretation and action recommendation.** Which disparity, if any, warrants a change — and what the range of changes is (retrain, recalibrate per group, adjust threshold, restrict deployment, add human review). Chapter 6 covers this space; do not omit it because it is uncomfortable.

## Concrete sketch

Using `numpy` and the confusion-matrix primitives from Chapter 4:

```python
import numpy as np
import pandas as pd

def group_confusion(y_true, y_pred, A):
    df = pd.DataFrame({"y": y_true, "yhat": y_pred, "a": A})
    rows = []
    for a, g in df.groupby("a"):
        n = len(g); n_pos = int((g["y"] == 1).sum()); n_neg = n - n_pos
        tp = int(((g["y"] == 1) & (g["yhat"] == 1)).sum())
        fp = int(((g["y"] == 0) & (g["yhat"] == 1)).sum())
        fn = int(((g["y"] == 1) & (g["yhat"] == 0)).sum())
        tn = int(((g["y"] == 0) & (g["yhat"] == 0)).sum())
        rows.append({
            "group": a, "n": n,
            "selection_rate": (tp + fp) / n if n else np.nan,   # for DP
            "tpr": tp / n_pos if n_pos else np.nan,             # for EO / EqOpp
            "fpr": fp / n_neg if n_neg else np.nan,             # for EO
            "ppv": tp / (tp + fp) if (tp + fp) else np.nan,     # for PP
            "tp": tp, "fp": fp, "tn": tn, "fn": fn,
        })
    return pd.DataFrame(rows)

def parity_gaps(df_groups, reference):
    ref = df_groups[df_groups["group"] == reference].iloc[0]
    gaps = df_groups.copy()
    for metric in ["selection_rate", "tpr", "fpr", "ppv"]:
        gaps[f"{metric}_diff_vs_ref"] = gaps[metric] - ref[metric]
        gaps[f"{metric}_ratio_vs_ref"] = gaps[metric] / ref[metric]
    return gaps
```

For calibration within groups, call the ECE / reliability-diagram machinery from Chapter 2 per group.

`fairlearn.metrics.MetricFrame` (Chapter 6) wraps all of this and computes the standard difference / ratio summaries directly.

## Common failure modes

- **Reporting one metric.** Any single fairness metric is easy to game and easy to satisfy at the expense of another. If the report has only demographic parity, ask for equalized odds; if it has only equalized odds, ask for calibration within groups.
- **No CIs.** With small subgroups, `1.6× demographic parity ratio` may or may not be sampling noise. Always CI.
- **Ignoring the operating point.** Fairness numbers computed at whatever threshold `predict()` uses are not the fairness numbers that ship. Re-measure at the operating point from Chapter 4.
- **Ignoring intersections.** Single-attribute reports hide intersectional harms.
- **Treating the sensitive attribute as clean data.** Inferred attributes (BISG, geographic proxies) have known measurement error that propagates into every fairness number computed from them. Characterize the error; do not pretend the attribute is ground truth.
- **Confusing measurement with mitigation.** This chapter is measurement. Actually equalizing the numbers is mitigation and comes with its own trade-offs (accuracy, calibration, policy). Chapter 6 covers the impossibility trade-offs; Fairlearn's mitigation algorithms are downstream of that discussion, not a substitute for it.
- **Publishing fairness gaps without a policy owner.** A fairness gap that no one is empowered to fix or defer is a liability. Name the owner in the report.

## Summary

Fairness measurement is per-slice measurement (Chapter 1) applied to protected attributes, plus a small number of standard definitions that quantify different senses of "the model treats groups the same." Demographic parity equalizes selection rates, equalized odds equalizes TPR and FPR, equal opportunity equalizes TPR alone, predictive parity equalizes precision, and calibration within groups equalizes the probabilistic meaning of the scores. Each requires a precisely-defined sensitive attribute (including how it was collected or inferred), CIs (mod-101 Chapters 3–4), a specific operating point (Chapter 4), and at least an intersectional look. Reporting one metric is not a fairness report. Chapter 6 covers the impossibility results — the reason you cannot generally satisfy all definitions at once — and the Fairlearn and Aequitas tooling that computes them end-to-end.
