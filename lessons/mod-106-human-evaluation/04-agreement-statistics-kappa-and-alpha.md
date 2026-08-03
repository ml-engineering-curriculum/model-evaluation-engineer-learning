# Agreement Statistics: Cohen's κ, Fleiss' κ, and Krippendorff's α

Every claim in this module about "reliable labels" rests on an agreement statistic. The pilot round reports κ. The calibration round's ship criterion is a κ or α threshold. The judge-vs-human calibration in mod-105 reports κ or a weighted κ against adjudicated humans. If you pick the wrong statistic — or read it without understanding what it corrects for and what it does not — the whole reliability story downstream is wrong. This chapter is the working knowledge you need to pick the right statistic per labelling shape, compute it with the right library, and read the resulting number honestly.

The three statistics you will actually use, in the field, are Cohen's κ, Fleiss' κ, and Krippendorff's α. Each answers a slightly different question. The most common failure is reporting one when the task calls for another, which produces a number that is either optimistic (usually) or arbitrary (occasionally). Weighted κ is a fourth variant that applies specifically to ordinal scales; it is not a separate statistic so much as a parameterization of κ.

## Why not just report percent agreement

"Annotators A and B agreed on 82% of items." That number is easy to compute and easy to explain. It is also, on any real task, misleading — because *some* of that agreement would have happened by chance even if both annotators were labelling at random. On a binary task with a 50/50 label distribution, two random annotators agree 50% of the time. On a binary task with a 90/10 skew (most items are the majority label), two random annotators agree 82% of the time — exactly the number above — while having zero *actual* agreement in any meaningful sense.

The purpose of κ and α is to *correct* observed agreement for what would have happened by chance. The general shape is:

```
κ = (p_o - p_e) / (1 - p_e)
```

where `p_o` is the observed agreement fraction and `p_e` is the chance-agreement fraction estimated from the marginal label distributions. If observed agreement equals chance, κ = 0. If observed agreement is perfect, κ = 1. Negative κ means annotators agreed *worse* than chance, which is either a data problem or a taxonomy problem worth investigating immediately.

α uses a very similar structure but is derived from *disagreement* rather than agreement, which lets it handle missing data and multiple raters cleanly.

## Cohen's κ: two raters, categorical labels

Cohen 1960. Two annotators, every item labelled by both, categorical labels (any number of categories). This is the default for a "let's have two people label everything and see how they agree" study — the pilot round from Chapter 3 with two annotators, or the judge-vs-human comparison from mod-105 where the "judge" is treated as one rater and the "human" as the other.

Chance agreement `p_e` is computed from each annotator's marginal distribution over labels. If both annotators label 90% of items as `helpful` and 10% as `not_helpful`, chance agreement is:

```
p_e = P(both say helpful) + P(both say not_helpful)
    = 0.9 * 0.9 + 0.1 * 0.1
    = 0.82
```

Which is why 82% observed agreement on a 90/10 skewed task produces κ = 0: you got exactly what chance would give you.

```python
from sklearn.metrics import cohen_kappa_score

# labels_a and labels_b: lists of the same length, aligned by item.
kappa = cohen_kappa_score(labels_a, labels_b)
```

**Two things κ does not do well.** First, it is *prevalence-sensitive*: on tasks with very skewed label distributions, κ can look bad even when annotators are agreeing meaningfully, because `p_e` is inflated. This is the "kappa paradox" (Feinstein and Cicchetti 1990) — high observed agreement, low κ, apparently contradictory. It is not a bug; it is κ telling you that on this label distribution most of the "agreement" would have happened by chance anyway. The response is to report both κ and observed agreement, or to switch to α (which is less prevalence-sensitive) as a cross-check.

Second, κ treats *all* disagreements as equally bad. On an ordinal scale where the labels are 1, 2, 3, 4, 5, a (1, 5) disagreement is much worse than a (2, 3) disagreement, but unweighted κ counts them the same. That is what weighted κ fixes.

## Weighted κ: ordinal labels

Cohen 1968. Two annotators, categorical labels *with an ordering*, and you want disagreements between distant categories to count more than disagreements between adjacent categories.

Weighted κ generalizes Cohen's κ by attaching a weight `w_ij` to every (annotator-A-label, annotator-B-label) cell of the confusion matrix. Two conventional weighting schemes:

- **Linear weights:** `w_ij = |i - j| / (k - 1)`. An adjacent-category disagreement (|i - j| = 1) counts as one step; a two-step disagreement counts as two.
- **Quadratic weights:** `w_ij = (i - j)² / (k - 1)²`. An adjacent disagreement counts as one; a two-step disagreement counts as four; a k-step disagreement counts as `(k - 1)²`.

Quadratic weights are the more common default for ordinal scales like Likert or star ratings — they match the intuition that "off by one" is nearly agreeing and "off by four" is essentially disagreeing.

```python
from sklearn.metrics import cohen_kappa_score

# For a 1-5 ordinal rubric:
weighted_kappa = cohen_kappa_score(
    labels_a, labels_b, weights="quadratic"
)
```

Weighted κ with quadratic weights is *always* at least as high as unweighted κ for the same data — the weighting is being generous to near-misses. Reporting both is useful: the gap tells you whether disagreements are mostly one-step (weighted κ close to unweighted → annotators are nearly-agreeing) or several-step (weighted κ much higher than unweighted → annotators are missing badly and unweighted κ is penalizing correctly).

## Fleiss' κ: three or more raters, categorical labels

Fleiss 1971. Extends the chance-corrected agreement idea to *multiple* raters. Notably, Fleiss' κ does not require the same set of raters on every item; it requires a *fixed number* of raters per item, drawn from a larger pool. If item 1 is labelled by raters (A, B, C) and item 2 is labelled by (D, E, F), both count.

The chance-agreement estimate `p_e` is computed from the marginal label distribution *pooled across all raters*, which is why Fleiss' κ does not require any specific rater to appear on every item — it treats the raters as interchangeable. This is often approximately what you want on a crowd platform where you cannot control who picks up which item.

```python
from statsmodels.stats.inter_rater import fleiss_kappa, aggregate_raters

# labels: a list of lists — labels[i] is the list of category labels
# assigned to item i by each of the N raters on that item.
# aggregate_raters converts to the (n_items x n_categories) counts
# matrix Fleiss' κ needs.
counts_matrix, category_names = aggregate_raters(labels)
kappa = fleiss_kappa(counts_matrix)
```

Fleiss' κ has the *same* prevalence sensitivity as Cohen's κ and does *not* have a weighted variant in the way Cohen's does. For ordinal scales with more than two raters and a need to weight disagreements, Krippendorff's α is the right tool.

## Krippendorff's α: any raters, any measurement level, missing data

Krippendorff 1970 onwards. This is the most general statistic in the family. It accepts:

- Any number of raters, possibly different on each item.
- Any *measurement level* — nominal (categorical), ordinal, interval, ratio — via a distance function on the label space.
- Missing data — items that some raters did not label.

The formula is `α = 1 - D_o / D_e`, where `D_o` is the observed disagreement (summed pairwise distances between rater labels on each item) and `D_e` is the expected disagreement under random chance. Nominal α with two raters and no missing data equals Cohen's κ up to a small-sample correction. Ordinal α with quadratic distances is closely related to weighted κ but generalizes to more raters.

```python
# krippendorff is a small, well-maintained Python package.
import krippendorff

# reliability_data: rows are raters, columns are items;
# use np.nan for missing labels.
alpha_nominal = krippendorff.alpha(
    reliability_data=data, level_of_measurement="nominal"
)
alpha_ordinal = krippendorff.alpha(
    reliability_data=data, level_of_measurement="ordinal"
)
```

α's practical advantages over κ:

- **Cleanly handles missing data.** Real annotation projects have annotators skip items, drop out mid-batch, or get filtered post-hoc. Cohen's κ needs pairs of complete labels; α does not.
- **Cleanly handles more than two raters** without requiring the fixed-N-per-item structure that Fleiss' κ needs.
- **Less prevalence-sensitive** on skewed distributions, though it is not immune.

Its practical disadvantages:

- Less well known outside of communication research and content analysis. Reviewers who ask for "kappa" may be confused by α unless you cite the mapping.
- Slightly more expensive to compute on large label matrices, though this rarely matters at eval-set scale.

Report α when: more than two raters, missing labels are non-trivial, or the label space has an interesting distance structure (e.g., "wrong by 30 seconds" is closer than "wrong by 10 minutes"). Report κ when: exactly two raters, complete data, and the audience expects κ.

## The picker: which statistic for which shape

A decision table for the common cases.

| Situation                                                              | Statistic                              | Library                            |
| ---------------------------------------------------------------------- | -------------------------------------- | ---------------------------------- |
| Two raters, binary or unordered categorical, complete data             | Cohen's κ                              | `sklearn.metrics.cohen_kappa_score` |
| Two raters, ordinal (1-5, Likert), complete data                       | Weighted κ (quadratic)                 | `sklearn.metrics.cohen_kappa_score(..., weights="quadratic")` |
| ≥ 3 raters, fixed N per item, categorical                              | Fleiss' κ                              | `statsmodels.stats.inter_rater.fleiss_kappa` |
| ≥ 3 raters, varying N per item OR missing labels, categorical          | Krippendorff's α (nominal)             | `krippendorff.alpha(level='nominal')` |
| ≥ 3 raters, ordinal, possibly missing data                             | Krippendorff's α (ordinal)             | `krippendorff.alpha(level='ordinal')` |
| Two raters, continuous or graded numeric scores                        | Spearman's ρ or Kendall's τ            | `scipy.stats.spearmanr`, `scipy.stats.kendalltau` |
| Pairwise verdicts (A / B / tie)                                        | Accuracy vs. adjudicated + κ if tie rate meaningful | `sklearn.metrics.accuracy_score` + `cohen_kappa_score` |
| Multi-label (any number of true labels per item)                       | Compute per-label κ; report mean and range | Loop `cohen_kappa_score` per label |
| Span selection (start-end indices over a passage)                      | F1 on span overlap, or Krippendorff's α with the appropriate distance | Custom / `krippendorff` |

Special notes:

- **On very skewed distributions**, report both κ and α, or κ and observed agreement. If they disagree, the disagreement is informative — usually κ is penalizing prevalence more than the reader expects.
- **Do not average kappas across items or slices.** The math does not compose the way an average does. If you want a per-slice number, compute κ per slice.
- **Bootstrap the CI on any reported κ.** A κ = 0.75 with a 95% CI of (0.68, 0.81) is a shippable measurement; the same point estimate with a CI of (0.55, 0.90) is not. `numpy.random.choice` over the item indices with 1,000–10,000 resamples is enough; the code is a few lines.

## How to read a κ number

The Landis and Koch 1977 scale is the most-cited interpretation:

- < 0.00: poor
- 0.00–0.20: slight
- 0.21–0.40: fair
- 0.41–0.60: moderate
- 0.61–0.80: substantial
- 0.81–1.00: almost perfect

Two caveats. First, these are *heuristic bands*, not statistical thresholds. Reviewers cite them because they exist, not because they are principled. Second, an appropriate κ threshold depends on the task's *stakes*, not on which band the number falls into: a κ = 0.5 might be perfectly acceptable for a low-stakes regression alerting metric that averages over many observations, and completely unacceptable for a safety-critical launch gate.

A useful practical framing:

- **κ ≥ 0.4 with a narrow CI:** the labels are *directional* — the ordering they produce is meaningful, but individual item labels should not be treated as ground truth without adjudication.
- **κ ≥ 0.6:** the labels can serve as ground truth for calibrating a judge, training a small model, or running a leaderboard.
- **κ ≥ 0.8:** the labels are strong enough for compliance-style claims where individual mislabels have real cost.

If your task consistently sits at κ = 0.3 across annotators and pilot revisions, the honest conclusion is often that the task as specified is not one humans can do reliably — and the fix is a taxonomy change, not another round of annotator training.

## A worked example end-to-end

Suppose two annotators label 200 items on a 3-way scale: `helpful`, `partially_helpful`, `not_helpful`. Observed marginal distributions come out close to 50/25/25 for each annotator. Observed agreement is 78%.

```python
from sklearn.metrics import cohen_kappa_score, confusion_matrix
import numpy as np

# 200 pairs of aligned labels
kappa = cohen_kappa_score(labels_a, labels_b)
# -> 0.66, say

# Bootstrap CI
n = len(labels_a)
rng = np.random.default_rng(0)
boot = []
for _ in range(1000):
    idx = rng.integers(0, n, n)
    boot.append(cohen_kappa_score(
        [labels_a[i] for i in idx],
        [labels_b[i] for i in idx],
    ))
lo, hi = np.percentile(boot, [2.5, 97.5])
# -> (0.58, 0.73), say

cm = confusion_matrix(
    labels_a, labels_b,
    labels=["not_helpful", "partially_helpful", "helpful"],
)
# Off-diagonals show WHERE the disagreements are.
```

You would report: "κ = 0.66 (95% CI 0.58–0.73) on a 200-item calibration set, two annotators, 3-way categorical scale. Confusion matrix indicates the majority of disagreement is between `partially_helpful` and `helpful`; annotators agree substantially on the `not_helpful` boundary." That is a two-sentence agreement summary with a matching diagnostic; it is enough for a downstream reader to trust or to challenge.

## Summary

Cohen's κ, Fleiss' κ, and Krippendorff's α are the three chance-corrected agreement statistics you will actually use. Cohen's κ handles two raters and categorical labels; weighted Cohen's κ (quadratic) handles two raters and ordinal labels. Fleiss' κ handles three-or-more raters with a fixed number per item but no weighting. Krippendorff's α handles arbitrary raters, missing labels, and ordinal or interval distance functions — it is the most general and the right default when the task shape is anything other than "two raters, categorical, complete." All of these should be reported with bootstrap 95% CIs and read against the task's stakes rather than against a universal threshold; on skewed distributions, report both κ and α (or κ and observed agreement) as a cross-check. The next chapter is what you do with the disagreements the agreement statistic surfaces: adjudication and gold-set rotation.
