# exercise-02: Bootstrap CI Implementation

**Estimated effort:** 3 hours

## Objective

Implement the percentile bootstrap and the paired bootstrap from scratch, validate your implementation against the reference implementations in `scipy.stats.bootstrap` and `statsmodels`, and produce a small library plus a report demonstrating correct behaviour on three eval metrics (accuracy, macro-F1, pairwise win-rate).

You are not simply calling a library function; you are building the routine and demonstrating you understand what it computes.

## Prerequisites

- Chapters 03 and 04 of this module.
- Python 3.11+, `numpy`, `scipy`, `statsmodels`, `scikit-learn`.
- Roughly one CPU-hour to run `B = 10000` bootstrap resamples over a handful of scenarios.

## Requirements

### Part A — implement

Write a small module `bootstrap_ci.py` with the following functions:

```python
def wilson_ci(successes: int, n: int, alpha: float = 0.05) -> tuple[float, float, float]:
    """Wilson score interval for a proportion. Returns (point, lo, hi).
    Do NOT call a library — implement the formula."""

def percentile_bootstrap_ci(
    data: np.ndarray,
    statistic_fn,
    B: int = 10000,
    alpha: float = 0.05,
    rng: np.random.Generator | None = None,
) -> tuple[float, float, float]:
    """Percentile bootstrap CI for a scalar statistic of a single-column sample."""

def paired_bootstrap_ci(
    outcomes_a: np.ndarray,
    outcomes_b: np.ndarray,
    statistic_fn,   # statistic_fn(x) -> scalar; called on each resample of A and B separately
    B: int = 10000,
    alpha: float = 0.05,
    rng: np.random.Generator | None = None,
) -> tuple[float, float, float]:
    """Paired bootstrap CI on the difference statistic_fn(B) - statistic_fn(A).
    Same resample indices for A and B."""
```

Requirements:

- Vectorize the resampling with `numpy.random.Generator.integers`. Do not write a Python `for` loop over `B`.
- Accept and use an injected `rng` for reproducibility. Default to a seeded generator when none is passed.
- `statistic_fn` is a user callable — support at least accuracy (mean of 0/1), macro-F1 (needs a 2-column input for `(y_true, y_pred)`), and per-item Likert averages.

### Part B — validate against reference implementations

Write `test_bootstrap.py` with these tests:

1. **Wilson equivalence.** Compare `wilson_ci(x, n)` against `statsmodels.stats.proportion.proportion_confint(x, n, method='wilson')` on ~20 `(x, n)` pairs covering small `n`, large `n`, and extreme `p̂`. Assert agreement to 1e-8.
2. **Percentile bootstrap equivalence.** For a fixed seed and a fixed `(data, statistic_fn)`, compare your percentile CI against `scipy.stats.bootstrap(data, statistic_fn, method='percentile', n_resamples=B, random_state=seed)`. Because RNG stream conventions differ, agreement will be up to the third decimal; document what tolerance you achieve and why.
3. **Paired bootstrap plausibility.** For a synthetic scenario where you know the true `P(B > A)` (e.g. `A ~ Bernoulli(0.5)`, `B = A ⊕ noise` calibrated so `P(B_i = 1) = 0.6`), verify that on many independent runs your 95% CI covers the true difference ~95% of the time. Coverage should be within `[0.93, 0.97]` on `≥ 500` synthetic runs.
4. **Clustering test.** Construct a dataset of 100 dialogues × 5 turns where within-dialogue turn outcomes are perfectly correlated. Show that resampling turns yields a CI that is *narrower* than resampling dialogues, and quantify the ratio. This is the "clustering ignored → CI too narrow" failure mode from Chapter 4.

### Part C — apply to three eval scenarios

Write `report.py` (or a notebook) that runs the three scenarios below and produces a short markdown table of point estimates and 95% CIs. Use publicly available or synthetic data — do not use proprietary or unlicensed data.

1. **Accuracy on a classification eval.** Use `sklearn.datasets.load_digits` or any public classification dataset. Train a small model, report accuracy with both a Wilson CI and a percentile bootstrap CI, and show they agree closely.
2. **Macro-F1 on the same eval.** Report macro-F1 with a percentile bootstrap CI (Wilson does not apply). Reproduce the CI by resampling `(y_true, y_pred)` pairs.
3. **Pairwise win-rate on a synthetic comparison.** Generate synthetic paired outcomes for two "models" where you control the true win-rate margin. Report the win-rate margin with a paired bootstrap CI and confirm it covers the true margin.

For each scenario, report the seed you used.

## Starter guidance

- Vectorized resampling in numpy: `idx = rng.integers(0, n, size=(B, n))` gives you a `B × n` index matrix; `data[idx]` broadcasts the resamples in one array operation.
- For metrics that need `(y_true, y_pred)` pairs (F1, MCC), pass a 2-column array as `data` and have `statistic_fn` accept the 2-column resample.
- `scipy.stats.bootstrap` uses a slightly different resampling convention (independent resamples per axis) than the simple `data[idx]` pattern. Read its docs before comparing.
- Coverage tests (part B, test 3) are slow — 500 runs × B=10000 resamples × n=200 items is ~10^9 operations. Vectorize aggressively; if you must, drop B to 2000 for the coverage sweep. Report what you did.

## Acceptance criteria

Your submission is acceptable if:

- All three functions have the exact signatures above and work on the shapes described.
- All four tests in Part B pass on a fresh clone with a documented seed. Test 3 documents its measured coverage.
- The Part C report has three point-estimate + CI rows, and the accuracy row shows Wilson and percentile-bootstrap CIs agreeing to at least two significant figures.
- Your code has no `for` loop over `B` (vectorization requirement).
- The README of your submission states the seed, `B`, and hardware / wall-clock for the coverage sweep.

## Stretch goals

- Add a **BCa** (bias-corrected and accelerated) implementation and show on a boundary-hugging statistic (e.g. accuracy near 1.0 on a small eval) where BCa's CI differs from percentile. Cite Efron & Tibshirani Ch. 14.
- Add a **cluster-bootstrap** function that takes an additional `cluster_ids` argument and resamples cluster IDs rather than individual observations. Use it to correctly re-run the clustering test from Part B.
- Add a **scorer-noise bootstrap**: given `(item, scorer_run)` outcomes, resample items with replacement and, for each resampled item, resample scorer runs with replacement (hierarchical bootstrap). Show that on a synthetic noisy-judge scenario, ignoring scorer noise produces a CI that undercovers.
