# exercise-01: Per-Slice Metrics Dashboard

**Estimated effort:** 3 hours

## Objective

Take a real, publicly available tabular classification dataset, train a baseline classifier, and produce the per-slice metrics report that Chapter 1 defines. The point is not to build a state-of-the-art model — it is to force through the *reporting* discipline: pick slicing axes with a stated reason, compute per-slice metrics with CIs, apply BH-FDR across the report, and package the output as an artifact a product reviewer could act on.

## Prerequisites

- mod-101 Chapters 3–6 (Wilson CIs, bootstrap, paired tests, BH-FDR).
- mod-103 Chapter 1.
- Python 3.11+ with `numpy`, `pandas`, `scikit-learn`, `statsmodels`, and a plotting library (`matplotlib` or `plotly`).

## Datasets (pick one)

All are publicly hosted and have documented slicing axes ready to use.

- **UCI Adult / Census Income** (Kohavi 1996). Binary classification (income > $50K). Natural slicing axes: `sex`, `race`, `native-country`, `education`, `age` bucketed, `workclass`. Available on the UCI ML Repository and via `sklearn.datasets.fetch_openml("adult", version=2)`.
- **COMPAS recidivism dataset** (ProPublica, Angwin et al. 2016). Binary classification (two-year recidivism). Slicing axes: `race`, `sex`, `age_cat`, `c_charge_degree`. Available from the ProPublica GitHub repository. Note: this dataset has a well-documented set of ethical and statistical criticisms; the point of using it here is to exercise the *reporting* discipline, not to endorse or dispute the underlying labels — cite the criticism in your report.
- **Bank Marketing** (Moro et al. 2014). Binary classification (term deposit subscription). Slicing axes: `job`, `marital`, `education`, `age` bucketed, `contact`. Available on the UCI ML Repository.
- **HMDA public loan data** (US Consumer Financial Protection Bureau; annual releases). Binary classification (loan approval). Real regulatory dataset with the slicing axes (`applicant_race`, `applicant_sex`, `applicant_ethnicity`, `income_bucket`, `state`) explicitly named in the disclosure schema.

If your organization has an internal classifier and eval set you are allowed to write about, you may substitute — the point of the exercise is the report, not the specific data.

## Requirements

### Part A — the slicing register (before you write any code)

Before you touch the data, write `SLICING_REGISTER.md` (~200–400 words). It must contain:

1. **The classifier's purpose in one sentence.** What would this model be used for; who acts on its output.
2. **The slicing axes**, 3–7 of them, each with a one-sentence reason drawn from:
   - **Product surface** — a distinction the product would treat differently.
   - **Risk surface** — a distinction where a failure carries asymmetric cost.
   - **Known-failure axis** — a slice where models on this task have historically failed (cite the source: paper, incident review, or a dataset-card note).
3. **Slice values per axis.** For categorical axes, enumerate the values including the `other` bucket. For numeric axes, state the bin boundaries and why they were chosen (product boundary, legal cutoff, quantile, etc.).
4. **Missing-value policy.** How null / unknown values are handled per axis (own slice vs. imputed).

The register is what you show a reviewer before they see the numbers. Its purpose is to prevent post-hoc slice fishing.

### Part B — train the baseline model

Train a baseline classifier on the chosen dataset. Keep this simple; the model is not the point of the exercise.

- **Model.** Logistic regression or a small gradient-boosted classifier (e.g. `sklearn.ensemble.HistGradientBoostingClassifier`).
- **Preprocessing.** Whatever handling you like for categoricals and numerics; document it in the writeup.
- **Split.** A single train/test split (typically 70/30 or 80/20, stratified on the label). Fix and document the seed. Do not tune hyperparameters on the test set.
- **Save artifacts.** Model pickle, split indices, and the predicted scores on the test set.

### Part C — the per-slice metrics table

Produce `per_slice_metrics.csv` (or Parquet) with one row per (`axis`, `slice`, `metric`) combination. Columns:

- `axis` — the slicing axis name.
- `slice` — the slice value (string).
- `n_slice` — count.
- `prevalence_positive` — base rate of the positive class in that slice.
- `metric_name` — one of `accuracy`, `precision`, `recall`, `f1`, `auc`.
- `metric_value` — point estimate.
- `ci_lower`, `ci_upper` — 95% CI (Wilson for accuracy / precision / recall; percentile bootstrap for F1 and AUC; mod-101 Chapters 3–4).
- `n_positive`, `n_negative` — for sanity checks.
- `sparse_flag` — `True` if `n_slice < 30` (or your documented threshold).
- `model_id` — a fixed identifier for the trained model.

Also produce `aggregate_metrics.csv` with the same columns but with `axis="aggregate"` and one row per metric, for the whole test set.

### Part D — the summary artifact

Write `SUMMARY.md`, ≤ 1 page. It must include:

1. **Aggregate performance** — accuracy, precision, recall, F1, AUC, each with CI.
2. **Top 3 worst-performing slices in absolute terms** (lowest F1 or lowest recall, your call — state and justify).
3. **Top 3 slices with the biggest gap from the aggregate** (i.e. the biggest relative underperformance).
4. **Sparse-slice callout** — slices with `sparse_flag=True` listed with `n_slice`. State whether they were suppressed, reported with a wide CI, or collapsed into `other`.
5. **A shipping recommendation** — ship / hold / conditionally ship on a subset — with the specific slice or slices driving each recommendation.

The summary is the artifact a product review reads. Do not restate the whole CSV in prose; do call out the numbers a decision-maker needs.

### Part E — the reproducibility harness

Write `run_dashboard.py` (or a Makefile) that regenerates all outputs from raw data given only a seed and the dataset choice. It must:

- Load the raw data from a public URL or a checked-in copy with documented provenance.
- Reproduce `per_slice_metrics.csv`, `aggregate_metrics.csv`, and `SUMMARY.md` byte-identically (except for timestamps) given the same seed.
- Emit a `MANIFEST.md` listing tool versions, data snapshot version, and the model_id.

### Part F — the paired-comparison variant (choose one of two)

Choose **one** of the following extensions to add a comparison dimension to the report:

- **F.1 — two-model comparison.** Train a second model (e.g. a random forest alongside the logistic regression, or the boosted tree with different depth), run the same per-slice report, and add columns `metric_diff`, `diff_ci_lower`, `diff_ci_upper`, `p_value`, `q_value_bh`. Use the paired bootstrap for the F1 / AUC diffs and McNemar for the accuracy diff (mod-101 Chapter 5). Apply BH-FDR across the entire report (mod-101 Chapter 6). Discuss any Simpson's-paradox slices — cases where the aggregate comparison and one or more slice comparisons disagree in direction.
- **F.2 — subgroup accuracy vs. calibration cross-check.** For each slice, compute both accuracy and per-slice ECE (mod-103 Chapter 2 preview — use `sklearn.calibration.calibration_curve` with `strategy="quantile"`). Identify slices where accuracy is high but calibration is poor (and vice versa). This is the setup for exercise-02.

## Starter guidance

- Do not skip Part A. A per-slice report without a stated reason for each axis is a spreadsheet with a plausible-looking column layout; the reason is what makes it defensible.
- Use `dropna=False` in your groupby operations so that missing values become their own slice rather than silently disappearing.
- For continuous axes (like `age`), pick bin boundaries that map to product or legal distinctions (`<18`, `18–24`, `25–34`, ...) rather than equal-width bins.
- For paired diffs on F1 and AUC, resample *examples* (paired bootstrap over the test set), not slices. Slices are a report-time grouping, not a sampling unit.
- `statsmodels.stats.multitest.multipletests(..., method="fdr_bh")` returns both the adjusted p-values (call them `q_value_bh` in your table) and the reject-or-not booleans. Use both.
- On the COMPAS dataset specifically, do not omit the ethics context; the dataset is heavily criticized and your report should note the criticism even though you are using the data for a reporting exercise.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- `SLICING_REGISTER.md` names 3–7 axes, each with a stated reason drawn from product, risk, or known-failure history.
- `per_slice_metrics.csv` has one row per (axis, slice, metric) combination, includes all required columns, and never has a silently missing value where `not_applicable` or `NaN with sparse_flag=True` would be more honest.
- Every metric point estimate is reported with a CI; no bare numbers.
- `SUMMARY.md` fits on one page, names the worst slices and the biggest-gap slices, calls out sparse slices, and ends with an actionable shipping recommendation.
- `run_dashboard.py` re-runs to byte-identical outputs given the seed.
- Part F is completed and adds a genuine second dimension (comparison or calibration cross-check) to the report.
- If the comparison in F.1 surfaces a Simpson's-paradox slice, it is called out explicitly in `SUMMARY.md`.
- BH-FDR is applied across the *reported* family, not per-axis (if F.1 was chosen).

## Stretch goals

- **Interactive dashboard.** Render the CSV as an interactive table (Streamlit, Panel, or a simple HTML/JS render). Filter by axis and by sparse_flag. A reviewer should be able to drill down without opening the CSV in Excel.
- **Production-population reweighting.** Simulate a different production population (e.g. the same test set with slice weights matched to a distribution you specify) and recompute the aggregate. Report both aggregates and discuss the composition-shift risk.
- **Slice-stability over time.** If your dataset has a time column, compute per-slice metrics for each of the last four quarters and report a time-series of per-slice performance. Flag slices that trended down.
- **Cross-comparison across three models.** Add a third model and switch from pairwise to a Kruskal–Wallis-style multi-model comparison, with BH-FDR across all pairwise contrasts. This is a controlled introduction to the family-wise problem in a way that scales to more than two arms.
