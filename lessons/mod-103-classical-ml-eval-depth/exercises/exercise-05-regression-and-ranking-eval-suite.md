# exercise-05: Regression and Ranking Eval Suite

**Estimated effort:** 2 hours

## Objective

Build a reusable evaluation harness that handles both a regression model and a ranking model, with subgroup breakdowns on both, and package the outputs as artifacts a product reviewer could read. The point is not to build a state-of-the-art model — it is to force through Chapter 7's discipline: pick multiple metrics for each problem class, always report a CI, apply subgroup breakdowns, and use the correct resampling unit (row-level for regression, query-level for ranking).

You will produce one harness that a colleague could point at any comparable dataset next quarter and get a defensible report from.

## Prerequisites

- mod-103 Chapters 1 and 7.
- mod-101 Chapter 4 (bootstrap CIs).
- Python 3.11+ with `numpy`, `pandas`, `scikit-learn`, `matplotlib`. Optional: `pytrec_eval` for ranking metrics that match the TREC reference implementation.

## Datasets (pick one from each column)

You will train (or load) one regression model and one ranking model. Reuse a dataset across the two problems only if it plausibly supports both.

### Regression dataset

- **California Housing** (`sklearn.datasets.fetch_california_housing`). Target: median house value. Natural slicing axes: `HouseAge` bucketed, `MedInc` bucketed, geographic quadrant derived from `Latitude` / `Longitude`.
- **NYC Taxi trip duration** (Kaggle competition `nyc-taxi-trip-duration` or the NYC TLC public trip records). Target: trip duration in seconds. Slicing axes: pickup borough, hour-of-day bucket, weekday vs. weekend, distance bucket. Large; downsample for the exercise.
- **Bike Sharing** (UCI `Bike Sharing Dataset`, Fanaee-T & Gama 2013). Target: hourly rental count. Slicing axes: season, weekday, weather category, hour-of-day bucket.
- **Superconductivity** (UCI, Hamidieh 2018). Target: critical temperature. Slicing axes: element-count bucket, mean-atomic-mass bucket. Small and clean.

### Ranking dataset

- **MovieLens 100K or 1M** (GroupLens). Task: rank held-out items per user by predicted rating. Graded relevance (ratings 1–5) or binarized (rating ≥ 4). Subgroup axes: user age bucket, user occupation, movie genre.
- **MSLR-WEB10K or MSLR-WEB30K** (Microsoft Learning to Rank). Task: rank documents per query with graded relevance labels 0–4. Subgroup axes: query length bucket, query frequency bucket if provided.
- **LETOR 4.0** (Microsoft Research learning-to-rank benchmark). Smaller alternative to MSLR with the same shape.
- **A hand-built synthetic ranking dataset** if the above are unavailable — 200 queries, 20 candidates each, with a known scoring rule and a small held-out perturbation. Acceptable only if you cannot install the real datasets.

If you have an internal dataset you are allowed to write about, substitute — the harness is the deliverable, not any specific dataset.

## Requirements

### Part A — the eval spec (before code)

Write `EVAL_SPEC.md` (~250–400 words). It must state, per problem:

1. **The model's product role** — what a downstream consumer would do with its output (a UI element, a business KPI, a resource allocation).
2. **The primary metric** — the one the shipping decision hangs on. Argue against the alternatives. For regression: RMSE, MAE, MAPE, R², sMAPE, or a specific quantile pinball loss — pick one and justify. For ranking: NDCG@k, MAP, MRR, Precision@k, or Recall@k — pick one and justify, and pick a specific `k`.
3. **The secondary metrics** — the ones you will still report as evidence. Chapter 7's point is that any single metric misleads; the report needs the paired context (e.g. RMSE alongside MAE to characterize the tails, NDCG@10 alongside NDCG@5 to characterize positional trade-offs).
4. **The slicing axes** for each problem — 2–4 per problem, each with a stated product-, risk-, or known-failure reason (Chapter 1).
5. **The sparse-slice policy** — the `n_slice` threshold below which a slice is suppressed or flagged.

### Part B — train (or load) the two models

- **Regression model.** A calibrated regressor: linear regression, `sklearn.ensemble.HistGradientBoostingRegressor`, or a quantile regressor if you chose pinball loss as the primary. Fixed seed. Train/test split (70/30 or 80/20).
- **Ranking model.** A pointwise regressor over `(query, candidate)` features (LambdaMART via `lightgbm`'s `LGBMRanker` is a solid default; `xgboost.XGBRanker` also works; a simple pointwise regressor over the query-item feature is acceptable for the exercise). Fixed seed. Train/test split by *query*, not by row — items from the same query must not straddle the split.

Save `y_true`, `y_pred` (regression) and the per-query `(scores, relevance_labels)` arrays (ranking), together with the slicing columns.

### Part C — the regression eval harness

Implement `regression_eval.py` exposing a function of the form:

```python
def regression_report(
    y_true, y_pred,
    slicing_columns=None,       # dict of {axis_name: array} for subgroup breakdowns
    metrics=("rmse", "mae", "mape", "r2"),
    n_boot=1000,
    sparse_threshold=30,
    seed=0,
): ...
```

Return an aggregate row and a per-slice table with:

- `axis`, `slice`, `n_slice`.
- One column per metric with a point estimate and 95% percentile bootstrap CI.
- `n_positive_y` / `y_mean` / `y_std` per slice so a reader can see when a slice has a target distribution unlike the aggregate.
- `sparse_flag` when `n_slice < sparse_threshold`.

Requirements on the implementation:

- Bootstrap the *rows* with replacement. Regression's resampling unit is the item.
- For MAPE, filter out rows where `|y_true| < ε` (state `ε` explicitly) or switch to sMAPE with a note. Do not silently divide by zero.
- If R² is negative on a slice, keep the number (do not clip) and flag the slice — a negative R² is a real finding, not a bug.
- If the primary metric is quantile loss, report empirical interval coverage alongside it (for `p50` / `p90` predictions, what fraction of `y_true` fell below the `p90` prediction?).

### Part D — the ranking eval harness

Implement `ranking_eval.py` exposing:

```python
def ranking_report(
    per_query,                  # list of dicts: {"q_id", "scores", "relevance", "slice_axes": {...}}
    metrics=("ndcg@10", "map", "mrr", "recall@10"),
    n_boot=1000,
    sparse_threshold=30,
    seed=0,
): ...
```

Return an aggregate row and a per-slice (query-level slicing) table with:

- `axis`, `slice`, `n_queries` (not items).
- One column per metric with a point estimate and 95% percentile bootstrap CI.
- Mean `|candidate_set|` and mean `|relevant|` per slice so the reader can see when a slice has an unusual candidate shape.
- `sparse_flag` when `n_queries < sparse_threshold`.

Requirements on the implementation:

- Bootstrap over **queries**, not items. Item-level bootstrap treats within-query items as exchangeable and reports the wrong CI (Chapter 7).
- For NDCG@k, use the standard `(2^rel − 1) / log2(i + 1)` gain / discount formulation and normalize per query. `sklearn.metrics.ndcg_score` is a convenient reference; verify it matches your implementation on a couple of hand-computed queries.
- For MAP with graded relevance, decide whether you binarize (typical: `rel ≥ threshold` becomes positive) or use graded AP, and document the choice.
- For queries with zero relevant items, define the per-query metric explicitly (typical: skip and note in the report; do not silently return 0).

Provide at least one hand-computed query in the tests so a reviewer can verify NDCG and MAP by inspection.

### Part E — the item-slice extension for ranking

Ranking's subgroup story is not only about the query. The *item* also has a slice — genre, popularity tier, freshness — and head-vs-tail item performance is often the more interesting number.

Add an **item-slice** report to `ranking_report` (or a second function). For each item slice, compute:

- The mean discounted exposure the model gives items in that slice (Chapter 7 mentions this as a fairness-adjacent quantity).
- The recall the model achieves on relevant items in that slice.
- Whether the slice is systematically under- or over-ranked relative to its share of the relevant set.

State whether your dataset supports this (MovieLens: yes, using movie genre; MSLR: possibly, via a bucketized feature). If not, note it and skip.

### Part F — the summary artifact

Write `EVAL_SUMMARY.md`, ≤ 2 pages, covering both models. For each:

1. **Aggregate performance** — the primary and secondary metrics with CIs.
2. **Baseline comparison** — regression against a naive predictor (mean, median, and last-value for time series) and ranking against a random-ranking baseline and a popularity-ranking baseline. Report the marginal improvement.
3. **Worst three slices in absolute terms** on the primary metric.
4. **Worst three slices relative to the aggregate** (the biggest gap).
5. **Sparse-slice callout** listing suppressed or wide-CI slices.
6. **A shipping recommendation** — ship / hold / conditionally ship on a subset — with the specific slice(s) driving each recommendation.

### Part G — the reproducibility harness

Write `run_eval.py` (or a Makefile) that regenerates all outputs given only a seed and the dataset choices. It must:

- Load the raw data from a public URL or a checked-in copy with documented provenance.
- Reproduce the aggregate table, the per-slice tables, and `EVAL_SUMMARY.md` byte-identically (except for timestamps).
- Emit a `MANIFEST.md` listing package versions, dataset snapshot, and the model_ids.

## Starter guidance

- `sklearn.metrics.ndcg_score` and `sklearn.metrics.average_precision_score` are the reference implementations for their respective metrics; sanity-check your hand-computed values against them.
- `pytrec_eval` wraps the standard TREC binary and is the tie-breaker if your numbers must match a published TREC baseline.
- For the regression bootstrap, `scipy.stats.bootstrap` (SciPy ≥ 1.7) is fully vectorized and about an order of magnitude faster than a Python loop.
- For the ranking bootstrap, `np.random.default_rng(seed).integers(0, n_queries, size=(n_boot, n_queries))` gives you the resample indices in one call; the inner loop is over the `n_boot` recomputations.
- Query-level train/test split is easy to get wrong. `sklearn.model_selection.GroupShuffleSplit(...).split(X, y, groups=q_id)` handles it correctly.
- Do not use `sklearn.metrics.mean_absolute_percentage_error` on data with `y = 0` items without filtering; the function returns `inf` and the mean collapses.
- For the item-slice exposure, be explicit about what "exposure" means (typical: cumulative `1 / log2(rank + 1)` over positions the item was shown at across queries).

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- `EVAL_SPEC.md` names the primary and secondary metrics per problem with reasons, and the slicing axes with product/risk/known-failure reasons.
- `regression_eval.py` and `ranking_eval.py` are usable as libraries — a colleague could import `regression_report` / `ranking_report` on a new dataset without reading the exercise.
- Every metric point estimate ships with a 95% bootstrap CI; there are no bare numbers in the report.
- Regression bootstrap resamples rows; ranking bootstrap resamples queries. This is checked in a docstring or comment where it matters.
- Per-slice tables include `n_slice` (regression) or `n_queries` (ranking), and sparse slices are flagged uniformly.
- The item-slice extension (Part E) is present or has a documented "not applicable to this dataset" note.
- `EVAL_SUMMARY.md` includes baseline comparisons and ends with an actionable shipping recommendation per model.
- `run_eval.py` reproduces to byte-identical outputs given the seed.

## Stretch goals

- **Quantile-regression coverage report.** For a quantile regressor predicting `p10` / `p50` / `p90`, report the empirical coverage of each prediction interval per slice; discuss where coverage is off and what that implies about the interval's usefulness.
- **Group-fairness for the regressor.** Report MAE and mean residual (`E[y − ŷ]`) per protected group; discuss whether the residual pattern suggests systematic over- or under-prediction for a group.
- **Rank-fairness diagnostics.** For a ranking model with item-side protected attributes, compute exposure disparity across item groups (Zehlike et al. 2017 FA*IR is the entry point). Report the disparity and where in the ranking it originates.
- **Position-bias-corrected metrics.** If your ranking labels are from clicks rather than editorial judgments, implement an inverse-propensity-weighted version of NDCG and compare against the naive NDCG. Discuss the magnitude of the correction.
- **Cross-model comparison.** Train a second ranking model (e.g. `LGBMRanker` alongside `XGBRanker`) and report the paired NDCG@10 difference per query with a paired-bootstrap CI (mod-101 Chapter 4), plus BH-FDR across slices (mod-101 Chapter 6).
- **Interactive report.** Render the per-slice tables as an interactive dashboard (Streamlit or Panel). A reviewer should be able to filter axes, toggle sparse-slice visibility, and download the underlying CSV.
