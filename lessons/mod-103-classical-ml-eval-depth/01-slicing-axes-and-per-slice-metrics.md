# Slicing Axes and the Per-Slice Metrics Report

Aggregate metrics hide the failures the eval was supposed to catch. A classifier at 0.92 accuracy overall can be 0.98 on the majority slice and 0.55 on a minority slice that happens to matter more per unit to the business — and the aggregate will move less than the CI you'd draw around it. The per-slice metrics report is the first defense: it forces the aggregate to explain itself against the axes that map to product surface and risk.

This chapter is about picking those axes, computing per-slice metrics defensibly, and packaging the result as an artifact a product team can act on. The estimator work — CIs on per-slice numbers, FDR correction across many slices — was covered in mod-101 Chapters 3–4 and Chapter 6; treat this as the "what slices, and how to lay them out" complement.

## Why aggregates lie

Two mechanisms show up repeatedly.

**Composition shift.** The test set is not the production population. If your test set is 90% English and production is 60% English, a per-language accuracy report is invariant to the shift; a headline "average accuracy" number is not, and it will drift silently every time the population changes. This is the specific reason single-number metrics disagree with production KPIs.

**Simpson's paradox.** Aggregate direction can reverse per-slice direction. Model A can beat Model B on every slice and lose on the aggregate, because Model A is disproportionately evaluated on hard slices where both models score low. This is not a curiosity. It reliably appears in A/B tests when the two arms have subtly different traffic composition (e.g. a new model was deployed first to a canary region). The per-slice report makes the paradox visible; the aggregate hides it.

**Rare-slice masking.** A model that fails catastrophically on a 2%-of-traffic slice can look fine at the aggregate because 98% of the traffic wasn't affected. The 2% is often the highest-value segment (enterprise customers, regulated regions, a specific safety-relevant intent). "Averaged over traffic" is the wrong average when the per-unit cost of failure is uneven.

The per-slice report addresses all three by forcing decision-makers to look at each slice on its own before they look at the aggregate.

## Choosing slicing axes: product surface, risk surface, and known-failure axes

Slicing is not "enumerate every column." A per-slice report with 200 axes is unreadable and the multiple-comparisons cost swamps the signal. The discipline is to pick a small set (typically 3–7 axes) whose slices actually change how you would act on the result.

Three sources.

**Product surface.** Any axis your product treats differently — a customer segment with a distinct SLA, a route in the app with distinct latency budgets, a locale with distinct legal exposure, a device class the UX branches on. If two segments get different treatment downstream, the metric needs to be reportable per-segment; otherwise the eval cannot answer "will this ship help or hurt segment X." Talk to the product owner; they can list these in 15 minutes.

**Risk surface.** Any axis where a failure carries asymmetric cost. Health-related content in a general chatbot, minors as an inferred user population, regulated jurisdictions, financial-advice-adjacent queries, safety-critical intents in an agent. The cost-asymmetry framing in Chapter 4 relies on this axis being visible in the eval. Legal, trust-and-safety, and compliance teams can enumerate them; do not enumerate them yourself and hope.

**Known-failure axes.** Where the model has failed before. Long inputs, non-Latin scripts, code-mixed language, images with low resolution, tabular rows with missing features, users with sparse history. If a bug report or an incident review named a slice, that slice becomes a permanent slicing axis until it is retired. Skipping this class is how the same failure ships twice.

Two axes that are frequently *wrong* to slice on:

- **Convenient-but-arbitrary bins** ("customers who signed up in the last 30 days" when 30 has no product meaning). Slices should match either an existing product distinction or a risk axis, not the analyst's Monday-morning intuition.
- **Sensitive attributes without a stated purpose.** Race, gender, and age slices *are* justified for a fairness measurement (Chapter 5), but should not be reported alongside performance slices without a fairness framing — otherwise you invite disparate-impact readings on numbers whose only purpose was diagnostic.

Write the slicing axes down in the eval spec. "Why does this axis exist?" should have a one-sentence answer that names product, risk, or a specific past incident.

## Slice definitions and coverage

For each axis, define the slice values explicitly, including a residual `other` bucket.

- **Categorical.** Enumerate the values. `locale ∈ {en-US, en-GB, es-ES, es-MX, ja-JP, other}`. Do not omit `other`; it collects the tail and lets you see if the tail is growing.
- **Ordinal / numeric.** Bin explicitly. `age ∈ {<18, 18–24, 25–34, 35–49, 50–64, 65+, unknown}`. Bin boundaries should match product boundaries where they exist (a legal cutoff, a subscription tier boundary), not equal-width bins for their own sake.
- **Latent slices.** Some slices are inferred at eval time (e.g. "queries containing code"). Document the inference rule — it becomes part of the eval config and needs to be versioned.
- **Missing values.** Always their own slice. If 15% of your test items have `country=null`, an `unknown` slice keeps that visible; imputing silently is how known-failure axes disappear.

Every item in the test set belongs to exactly one slice value per axis. Every axis is decomposable into non-overlapping slice values that sum to the full test set (a partition). This is the invariant your report checker enforces before it renders.

## Per-slice metrics: what to compute

For a classifier, per slice, at minimum:

- **Sample count** `n_slice`. Report before anything else. All the numbers below are meaningless without it.
- **Prevalence** — the base rate of the positive class in that slice. Different from `n_slice`; a slice can be big and low-prevalence. Accuracy on a 1%-prevalence slice at the trivial predictor `0` is 99%, and that shape has to be visible in the report.
- **The metric of interest** — accuracy, precision, recall, F1, AUC — whichever the eval's headline is. Same metric across all slices.
- **Confidence interval on the metric.** Wilson interval for proportions (accuracy, precision, recall); percentile or paired bootstrap for F1 and AUC. See mod-101 Chapters 3–4. Never report a per-slice point estimate without its CI.
- **The metric on the complement** — the metric on "everyone not in this slice." Lets a reader see the gap and compare like-for-like without redoing the arithmetic.

For a comparison of two models, per slice, add:

- **Paired difference** in the metric, with a paired-bootstrap CI (mod-101 Chapter 4). Not the difference of independent CIs — paired.
- **The paired-test p-value** (McNemar for accuracy on binary tasks, paired bootstrap for other metrics; mod-101 Chapter 5). Adjust with Benjamini–Hochberg across all slices, not per-axis (mod-101 Chapter 6).

Do not report metrics that require classes the slice does not contain. A slice with zero positives cannot have a recall; a slice with all-positives cannot have a specificity. Print the metric as `N/A (n_pos=0)` rather than silently returning 0 or NaN — the latter propagate into means that make no sense.

## Minimum sample size per slice: what to do about small slices

Per-slice numbers become uninformative below some `n_slice`. Reporting `recall = 0.50 (95% CI [0.10, 0.90])` on a slice of `n_slice=4` is technically correct and operationally useless. Two defensible approaches:

- **Report with a wide CI and a size flag.** The interval width self-limits the interpretation. Add a `data_quality: sparse` flag so a downstream reader can filter. The Wilson interval is well-behaved even at very small `n`; the point estimate is not, but the CI honestly says so.
- **Suppress and note.** For very small slices (say `n_slice < 30`), collapse into an `other` bucket or report `n < threshold, not reported`. This is standard in demographic reporting where publishing small-cell counts is a privacy risk in addition to a statistical risk.

Pick a policy and apply it uniformly across the report. Ad-hoc suppression looks like cherry-picking even when it is not.

If a slice is important but chronically small, the right response is to oversample it into the *eval* (not into training). "Add 200 items in the sparse slice on the next collection cycle" is a concrete acceptance criterion for the benchmark maintainer.

## Multiple comparisons across many slices

A per-slice report with 6 axes and an average of 5 slice values per axis is 30 numbers per metric per model, and 30 paired-difference tests when comparing two models. Under the null, ~1.5 of those tests will be significant at α=0.05 by chance.

Apply Benjamini–Hochberg FDR (mod-101 Chapter 6) across the full family of slice tests you actually report. The correction is over the report, not over the axis — the reader will see the whole report at once, so all 30 tests are the family. Under-correcting produces false discoveries you will then have to walk back; over-correcting (Bonferroni across the report) sacrifices power for false-positive control you do not need.

Do *not* correct for tests you did not report or plan to report. If you computed 300 candidate slices and reported the 30 with the smallest p-values, correcting only over those 30 is dishonest — you have to correct over the 300 you selected from. The clean fix is to declare the report structure in the spec, not to pick slices after seeing the results.

## The Simpson's paradox check

Whenever an aggregate comparison points one way and one or more slice comparisons point the other way, stop and explain before shipping the aggregate as the headline. Two things to do:

1. **Recompute the aggregate weighted by production traffic** rather than test-set traffic. If the test-set-weighted aggregate says "Model B is better" and the production-weighted aggregate says "Model A is better," the eval population is different from production and that has to be in the report as a validity caveat (mod-101 Chapter 2).
2. **Report per-slice metric plus a slice-weight column.** A reader can then reweight in their head. If the slice weights are wildly different between test and production, the report says so.

Do not "fix" the paradox by dropping slices until the aggregate agrees with them. Report the paradox; let a decision-maker choose the population they care about.

## The report artifact: what to hand a product team

The output of the per-slice analysis is a machine-readable table plus a human-readable summary.

**The table (CSV, Parquet, or JSONL).** Columns:

- `axis` — the slicing axis name.
- `slice` — the slice value.
- `n_slice` — count.
- `prevalence_positive` (if applicable).
- `metric_name` — one row per metric per slice; long form is easier to filter than wide.
- `metric_value`, `ci_lower`, `ci_upper` — always together.
- `n_positive`, `n_negative` (if applicable) — so a reader can spot degenerate slices.
- `model_id` and `benchmark_version` (the three-hash version from mod-102 Chapter 6) — so a reader knows which run this row belongs to.

For a two-model comparison, add `metric_diff`, `diff_ci_lower`, `diff_ci_upper`, `p_value`, `q_value_bh` (the BH-adjusted p-value).

**The summary (a page of markdown).** For a shipping decision:

1. Aggregate metric with CI, model IDs, benchmark version.
2. Top 3 slices where the model performs worst in absolute terms.
3. Top 3 slices where the model performs worst *relative to the aggregate* (i.e. the biggest gap).
4. In a two-model comparison: any slice where the comparison's sign is opposite the aggregate's sign, called out explicitly.
5. Any slice with `n_slice < threshold` listed with the sparsity flag.
6. The action recommendation — ship / hold / conditionally ship on a subset — with the specific slice or slices driving each recommendation.

The summary is the artifact a product review reads. The table is what an analyst re-runs a query against. Ship both; do not ship only the aggregate.

## Concrete sketch

Assume a binary classifier, a test set with columns `y_true`, `y_pred`, `y_score`, and slicing columns `locale`, `device`, `age_bucket`. In `pandas` / scikit-learn:

```python
import numpy as np
import pandas as pd
from sklearn.metrics import precision_score, recall_score, f1_score
from statsmodels.stats.proportion import proportion_confint

def wilson(k, n, alpha=0.05):
    if n == 0:
        return (np.nan, np.nan)
    return proportion_confint(k, n, alpha=alpha, method="wilson")

def per_slice_row(group, axis, slice_value):
    n = len(group)
    y_true = group["y_true"].to_numpy()
    y_pred = group["y_pred"].to_numpy()
    n_pos = int(y_true.sum())
    tp = int(((y_true == 1) & (y_pred == 1)).sum())
    fp = int(((y_true == 0) & (y_pred == 1)).sum())
    fn = int(((y_true == 1) & (y_pred == 0)).sum())
    # Precision and recall as proportions -> Wilson interval.
    p_ci = wilson(tp, tp + fp) if (tp + fp) else (np.nan, np.nan)
    r_ci = wilson(tp, tp + fn) if (tp + fn) else (np.nan, np.nan)
    return {
        "axis": axis, "slice": slice_value, "n_slice": n,
        "prevalence_positive": n_pos / n if n else np.nan,
        "precision": tp / (tp + fp) if (tp + fp) else np.nan,
        "precision_ci_lower": p_ci[0], "precision_ci_upper": p_ci[1],
        "recall": tp / (tp + fn) if (tp + fn) else np.nan,
        "recall_ci_lower": r_ci[0], "recall_ci_upper": r_ci[1],
        "n_positive": n_pos, "sparse_flag": n < 30,
    }

def per_slice_report(df, axes):
    rows = []
    for axis in axes:
        for slice_value, group in df.groupby(axis, dropna=False):
            rows.append(per_slice_row(group, axis, str(slice_value)))
    return pd.DataFrame(rows)
```

F1 needs a bootstrap (mod-101 Chapter 4); wrap it the same way. For two-model comparisons, replace the loop body with a paired-bootstrap of the difference on each slice, then run Benjamini–Hochberg on the p-value column with `statsmodels.stats.multitest.multipletests(..., method="fdr_bh")`.

## Summary

The per-slice metrics report is the single most cost-effective eval artifact for classical ML systems: it exposes the composition and rare-slice failures the aggregate hides, defends against Simpson's paradox, and gives a product team the vocabulary to argue about ship vs. hold on the actual segment they care about. Pick slicing axes from the product surface, the risk surface, and the known-failure history — 3–7 axes total, each with a documented reason. Enumerate slice values including an `other` bucket, handle missing values as their own slice, and report every per-slice number with a Wilson or bootstrap CI. Correct multiple comparisons across the report with Benjamini–Hochberg. Ship a machine-readable table plus a one-page markdown summary that names the worst slices, the sign-flip cases, and the action recommendation. Chapter 2 turns to the second question the per-slice report cannot answer on its own: whether the model's *probabilities* are trustworthy.
