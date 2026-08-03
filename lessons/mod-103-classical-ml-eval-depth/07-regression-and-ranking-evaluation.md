# Regression and Ranking Evaluation with Subgroup Breakdowns

Classification metrics are the ones evaluation folklore covers first, but a large fraction of classical ML in production is regression (predicting a numeric quantity — price, ETA, dwell time, LTV, energy consumption) or ranking (ordering a candidate set — search, recommendations, ad targeting). Their metrics have their own failure modes, and the per-slice / subgroup-breakdown discipline from Chapter 1 applies just as forcefully. This chapter is the compact reference for the metrics you will actually ship and the shapes of report they belong in.

## Regression metrics: the standard four and when to use them

### RMSE — root mean squared error

`RMSE = sqrt( (1/n) · Σ (y_i − ŷ_i)² )`.

Same units as the target. Penalizes large errors quadratically. Sensitive to outliers because of the square.

- **Use RMSE when large errors are disproportionately bad** — a delivery ETA off by an hour is much worse than four ETAs off by 15 minutes.
- **Do not use RMSE when outliers are noise** — a single labelled outlier can dominate the metric.

Reported with a bootstrap CI (mod-101 Chapter 4). RMSE is not a proportion, so Wilson does not apply.

### MAE — mean absolute error

`MAE = (1/n) · Σ |y_i − ŷ_i|`.

Same units as the target. Penalizes errors linearly. Robust to outliers. Interpretation is direct: "on average, our prediction is off by ±MAE units."

- **Use MAE when the loss on the business side is roughly linear** — every dollar off on a price prediction costs one dollar of margin.
- **Use MAE when outliers are noise** — quantile-based robustness is what you get.
- **MAE and RMSE together tell you about outliers.** If RMSE >> MAE, the error distribution has heavy tails; if RMSE ≈ MAE, the errors are close to uniform.

### MAPE — mean absolute percentage error

`MAPE = (100 / n) · Σ | (y_i − ŷ_i) / y_i |`.

Unit-free (percentage). Interpretable across problems with different target scales.

- **Use MAPE when relative error is what matters** — a `$10` error on a `$100` price is worse than a `$10` error on a `$10000` price.
- **Do not use MAPE when `y` can be zero or near-zero** — the denominator blows up and one small-`y` item dominates the metric.
- **MAPE is asymmetric.** It penalizes under-prediction more than over-prediction (given the same absolute error): if `y = 100` and `ŷ = 50`, MAPE contribution is `50%`; if `y = 100` and `ŷ = 150`, MAPE contribution is `50%` too, but if `y = 50` and `ŷ = 100`, MAPE contribution is `100%`. When `y` is right-skewed, MAPE systematically prefers models that under-predict.
- **sMAPE** (symmetric MAPE) partially addresses the asymmetry: `sMAPE = (100 / n) · Σ |y_i − ŷ_i| / ((|y_i| + |ŷ_i|)/2)`. Not a full fix; still sensitive near zero.

### R² — coefficient of determination

`R² = 1 − (Σ (y_i − ŷ_i)²) / (Σ (y_i − ȳ)²)`.

Unit-free. Interpretable as "fraction of variance explained relative to the mean baseline."

- **R² = 1** — perfect prediction.
- **R² = 0** — predicts no better than the mean of `y`.
- **R² < 0** — predicts *worse* than the mean baseline. Possible; happens frequently on held-out data with severe distribution shift.
- **Use R² when comparing across problems** — same scale (0 to 1 in the well-behaved case) even when target ranges differ.
- **Do not use R² alone in production reporting** — it does not tell a business owner how many dollars off the prediction is. Pair with RMSE or MAE.

### Quantile loss (pinball loss)

`L_τ(y, ŷ) = max( τ · (y − ŷ), (τ − 1) · (y − ŷ) )` for a target quantile `τ ∈ (0, 1)`.

Proper scoring rule for the `τ`-quantile of `y | x`. Use when the model is predicting a specific quantile (e.g. `p50` and `p90` ETA predictions), not the mean. The mean of `L_τ` over the test set is the standard evaluation for quantile regression.

If your model predicts multiple quantiles, compute one pinball loss per quantile and report them together; also check **calibration in the interval sense**: for a nominal `90%` prediction interval, verify that the empirical coverage is close to `90%`.

## Regression evaluation caveats

- **Report multiple metrics.** RMSE alone tells you about outliers; MAE alone about typical error; MAPE alone about relative error; R² alone about baseline improvement. Any one is misleading in isolation.
- **Bootstrap CIs everywhere.** No point estimate without an interval. Regression metrics can be unstable at small `n` and with heavy-tailed error distributions.
- **Report metrics per prediction magnitude.** A single MAE can hide poor performance on small values (a common problem for MAPE-optimized models). Bin `y` by decile and report metrics per decile.
- **Baseline against a naive predictor.** At minimum, predict the mean, the median, and (for time series) the previous value. If your model beats "predict the mean" by 3%, the R² is 0.03, and you may not have a shippable model.
- **Match the metric to the loss you trained on.** A model trained with squared error will have a lower RMSE than one trained with absolute error, on the same data, because it was optimized for it. Cross-comparisons should either use a metric neither model was directly trained on or report both losses.

### Subgroup breakdowns for regression

The per-slice framework from Chapter 1 applies. For each slice:

- Report `n_slice`, MAE, RMSE, MAPE (if `y > 0`), and R² with bootstrap CIs.
- Report the distribution of `y` and of `ŷ` per slice — a slice with very different target statistics will move all the metrics, and the report should show that.
- Cross-check for pathologies: slices where R² is negative are cases where the model performs worse than the slice mean; slices with much higher MAE than the aggregate are the ones that will bring production KPIs down when the population weight in that slice grows.

Fairness for regression models has a smaller literature than for classification, but the analogues are directly usable: **error disparity** (MAE_a − MAE_a' across groups), **prediction distribution disparity** (KS statistic between `ŷ | A = a` and `ŷ | A = a'`), and **calibration by group** (does the residual conditional mean equal zero within each group?). Report at least MAE and mean-residual per protected attribute.

## Ranking metrics: what is being measured

Ranking evaluations start from a different data shape than classification or regression. Instead of `(item, label)` pairs, the data is:

- A set of **queries** `q` (a search query, a user session, an ad request).
- Per query, a **candidate set** of items with **relevance labels** `rel_i ≥ 0` (often graded: `0` = irrelevant, `1` = fair, `2` = good, `3` = excellent; or binary: `0`/`1`).
- The **model's predicted ranking** of those candidates — a permutation.

The metrics evaluate how well the predicted ranking places high-relevance items near the top.

### Precision@k and Recall@k

- **Precision@k** — fraction of the top-`k` predicted items that are relevant. `P@k = |top-k ∩ relevant| / k`.
- **Recall@k** — fraction of relevant items that appear in the top-`k`. `R@k = |top-k ∩ relevant| / |relevant|`.

Simple, interpretable. Pick `k` based on the product surface — `k = 3` for a "top 3 suggestions" UI, `k = 10` for a page of results.

Not sensitive to the order *within* the top-`k`. A ranking that puts the best result at position `k` scores the same as one that puts it at position `1`. When the position matters, use MAP or NDCG.

### MAP — mean average precision

**Average precision** per query is the mean precision computed at every rank at which a relevant item appears:

`AP(q) = (1 / |relevant|) · Σ_{k : rel_k = 1} P@k`.

**Mean average precision** is the mean over queries: `MAP = (1 / |Q|) · Σ_q AP(q)`.

- Uses binary relevance labels.
- Rewards ranking relevant items early; penalizes putting them late.
- Bounded in `[0, 1]`; a perfect ranking gets `1.0`.

### NDCG — normalized discounted cumulative gain

For graded relevance labels, NDCG is the standard.

**Discounted cumulative gain** at rank `k` for a query:

`DCG@k(q) = Σ_{i=1..k} (2^{rel_i} − 1) / log2(i + 1)`.

**Ideal DCG** `IDCG@k(q)` — the DCG of the ideal ranking (relevant items sorted by relevance desc).

**NDCG@k(q) = DCG@k(q) / IDCG@k(q)**.

**NDCG@k = mean over queries.**

- Bounded in `[0, 1]` after normalization; the mean is a reportable single number.
- Handles graded relevance (`2^rel − 1` gain rewards excellent-labelled items more than good-labelled items).
- Positional discount (`1 / log2(i + 1)`) means an excellent item at rank 1 counts more than at rank 5.
- Standard `k` values are `5`, `10`, `20`. Match to the product.

### MRR — mean reciprocal rank

For each query, take the reciprocal of the rank of the first relevant item. Average over queries.

`MRR = (1 / |Q|) · Σ_q (1 / rank_q_first_relevant)`.

Simple. Use when the task is "find one good answer as early as possible" (e.g. QA retrieval where the first correct passage is what matters). Uninformative when there are many relevant items per query and the tail matters.

### Hit rate / recall@1

For a `top-1` recommendation task, hit rate `H@1` is just "did the top item hit the relevant set." Report it alongside NDCG@k when the product surfaces only the top result to the user (a single autocomplete suggestion, a default recommendation, a single voice-assistant response).

## Ranking evaluation caveats

- **Relevance labels are expensive and noisy.** Every ranking metric assumes the `rel` labels are trustworthy. Use the annotation-quality discipline from mod-102 Chapter 3–4 (pilots, IAA, adjudication) to make them so. Cheap, noisy relevance labels produce misleadingly stable-looking metrics.
- **Position bias in implicit-feedback labels.** If your `rel` labels come from clicks or impressions, the labels themselves are biased by the current model's ranking (users click items shown at the top). Corrections include inverse-propensity weighting, click model estimation (e.g. cascade models), or explicit relevance labels; the choice is a whole subfield. Skipping the correction means the offline eval systematically reproduces the current ranker's biases.
- **Comparing rankings requires the same candidate sets.** Two rankers evaluated on differently-sourced candidate sets are not comparable. Fix the candidate set (e.g. `top-100` from a shared first-stage retriever) before ranking-stage evaluation.
- **NDCG's positional discount is arbitrary.** `log2(i + 1)` is convention, not truth. If your product surfaces items very differently (a grid of thumbnails vs. a linear list), NDCG's positional weights may not match user-perceived importance.
- **Metrics on different `k` values move differently.** A model that wins on NDCG@5 can lose on NDCG@20 by favouring a strong top-3 at the cost of tail relevance. Report multiple `k`s when the product surfaces more than one page.
- **CIs.** Query-level bootstrap (resample queries with replacement) is the correct resampling scheme. Item-level bootstrap is wrong; items within a query are not exchangeable.

### Subgroup breakdowns for ranking

Applying Chapter 1 to ranking metrics:

- **Slice on the query.** Query language, query length, query type (navigational / informational / transactional), user segment, time-of-day.
- **Slice on the item.** Category, price tier, freshness, popularity bucket. Head-vs-tail item performance is nearly always different, and often the tail is where the product needs the ranker to help most.
- **Slice on the intersection.** Head-query × tail-item performance is a distinct metric from head-query × head-item; both matter.

Per slice: NDCG@k, MAP, MRR (whichever is the headline), each with a query-level bootstrap CI.

Fairness in ranking has additional definitions specific to exposure and allocation — e.g. **exposure disparity** (average discounted exposure difference across item groups), and **rank-aware disparate impact**. Zehlike et al. (2017) FA*IR and the Diaz et al. (2020) evaluation-of-stochastic-rankings work are the entry points; the general shape is that ranking fairness is about how much *screen real estate* different groups' items get, not just about label accuracy.

## Concrete sketch

Regression metrics from scikit-learn:

```python
import numpy as np
from sklearn.metrics import (
    mean_squared_error, mean_absolute_error,
    mean_absolute_percentage_error, r2_score,
)

def regression_report(y_true, y_pred):
    return {
        "rmse": float(np.sqrt(mean_squared_error(y_true, y_pred))),
        "mae": float(mean_absolute_error(y_true, y_pred)),
        "mape": float(mean_absolute_percentage_error(y_true, y_pred)),
        "r2": float(r2_score(y_true, y_pred)),
        "n": int(len(y_true)),
    }
```

Ranking metrics from scikit-learn:

```python
from sklearn.metrics import ndcg_score, average_precision_score

# y_true: (n_queries, n_candidates) matrix of relevance labels
# y_score: (n_queries, n_candidates) matrix of predicted scores
ndcg_at_10 = ndcg_score(y_true, y_score, k=10)
map_at_10 = np.mean([
    average_precision_score(y_true[i], y_score[i]) for i in range(len(y_true))
])
```

For MRR, `Recall@k`, and more complex evaluations, `pytrec_eval` (a Python wrapper for the standard `trec_eval` binary) is the reference implementation with the same numbers TREC papers report. If you compare against a published TREC baseline, use `pytrec_eval` — hand-rolled metrics do not always match `trec_eval` at the edge cases.

For query-level bootstrap on ranking metrics:

```python
def bootstrap_ndcg(y_true, y_score, k=10, n_boot=1000, seed=0):
    rng = np.random.default_rng(seed)
    n = len(y_true)
    values = []
    for _ in range(n_boot):
        idx = rng.integers(0, n, size=n)   # resample queries with replacement
        values.append(ndcg_score(y_true[idx], y_score[idx], k=k))
    return np.percentile(values, [2.5, 50, 97.5])
```

## Common failure modes

- **Reporting a single regression metric.** Almost always misleading. Report RMSE + MAE (or MAE + MAPE) as a minimum pair.
- **Reporting R² without a baseline.** `R² = 0.7` sounds great until the naive-mean baseline gives `R² = 0.65`; the useful metric is the marginal improvement.
- **Using MAPE with near-zero `y`.** One tiny-`y` item can produce a MAPE > 1000%. Filter or switch metric.
- **Position-bias-uncorrected ranking metrics on click labels.** The offline eval will over-fit to the current model. Correct or acknowledge.
- **Item-level bootstrap on ranking metrics.** Wrong. Query-level.
- **NDCG on binary labels.** Not wrong, but MAP is more standard for binary relevance and easier to explain. Save NDCG for graded labels.
- **Comparing rankings on differently-sourced candidate sets.** Not comparable. Fix the candidate set first.

## Summary

Regression and ranking metrics carry the same discipline as the classification metrics in earlier chapters: pick metrics that match the business's loss function, report multiple metrics rather than a single one, always include bootstrap CIs, and apply the per-slice / subgroup discipline from Chapter 1. For regression, RMSE and MAE together characterize the shape of the error distribution, MAPE handles cross-scale interpretability at the cost of near-zero fragility, and R² is a baseline-improvement measure that needs a baseline to be meaningful. For ranking, NDCG@k with graded relevance and MAP with binary relevance are the standard; MRR handles first-hit tasks. Query-level bootstrap is the correct CI for ranking. Both problem classes deserve subgroup breakdowns on the product-surface, risk-surface, and known-failure axes, and both have their own fairness variants worth reporting when protected attributes are in play. With Chapters 1–7, the classical-ML evaluator has the toolkit; downstream modules (LLM, judge, agent) reuse these primitives against much noisier score distributions and much less-well-defined tasks.
