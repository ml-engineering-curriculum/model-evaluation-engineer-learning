# Bootstrap Confidence Intervals for F1, Averages, and Win Rates

Wilson works for a proportion. For most other eval metrics — F1, macro-averaged accuracy, ROUGE, per-item Likert averages, ELO gaps — there is no equally simple closed-form CI. The bootstrap gives you a general, defensible CI for anything that is a function of the sample.

This chapter covers the two variants you will use almost every day: the **percentile bootstrap** for a single-system statistic, and the **paired bootstrap** for a comparison between two systems.

## The bootstrap in one sentence

Given a sample of size `n`, the bootstrap approximates the sampling distribution of a statistic by repeatedly resampling `n` items *with replacement* from the observed sample and recomputing the statistic on each resample.

Efron (1979) introduced the method; Efron & Tibshirani (1993) and Davison & Hinkley (1997) are the standard book-length treatments. For most eval work, the mechanics reduce to a short loop.

## Percentile bootstrap for a single system

Given per-item outcomes `x = (x_1, ..., x_n)` and a statistic `θ̂ = T(x)`:

    for b in 1..B:
        resample x* of size n from x, with replacement
        θ̂_b = T(x*)
    sort {θ̂_b}
    CI = [ percentile(2.5), percentile(97.5) ]

Choices to make explicitly:

- **`B` — number of resamples.** `B = 1000` gives you a percentile CI accurate to about a percentage point; `B = 10000` is what to use in a report. For very small `α` (0.001) you need more resamples in the tails. Bootstrap is embarrassingly parallel; there is little reason to skimp.
- **What is a resample unit?** For an eval where each item is independent, resample items. For grouped or clustered evals (multi-turn dialogues, multi-question passages, multiple annotators per item), resample *groups*, not individual observations. Failing to cluster inflates the effective N and shrinks the CI. See the "gotchas" section below.
- **Which CI variant?** Percentile is the default. BCa (bias-corrected and accelerated) is more accurate for skewed statistics; use it when a statistic is bounded and near the boundary. `scipy.stats.bootstrap(..., method='BCa')` gives you BCa out of the box.

Worked mini-example. Suppose your per-item F1 scores on `n = 300` items give a macro-F1 of `0.712`. A percentile bootstrap with `B = 10000` might yield a 95% CI of roughly `[0.685, 0.738]`. That is the number you ship: not `0.712`, but `0.712 (95% CI [0.685, 0.738])`.

Skeleton implementation with `numpy`:

    import numpy as np

    def percentile_ci(scores, statistic_fn, B=10000, alpha=0.05, rng=None):
        rng = rng or np.random.default_rng(0)
        n = len(scores)
        idx = rng.integers(0, n, size=(B, n))
        boot = np.array([statistic_fn(scores[i]) for i in idx])
        lo, hi = np.percentile(boot, [100*alpha/2, 100*(1 - alpha/2)])
        return statistic_fn(scores), lo, hi

For `scipy` users, `scipy.stats.bootstrap` handles the resampling and the CI computation, including BCa; prefer it in library code so you get vectorized resampling and the BCa correction.

## Percentile bootstrap for F1: what changes

F1 is the harmonic mean of precision and recall. It is *not* the mean of per-item F1s (that would be a different statistic), and you cannot bootstrap it by resampling per-item F1 scores. You have to resample the per-item `(y_true, y_pred)` pairs and *recompute* F1 on each resample:

    for b in 1..B:
        resample (y_true, y_pred) pairs of size n, with replacement
        compute precision_b, recall_b on the resample
        F1_b = 2 * precision_b * recall_b / (precision_b + recall_b + eps)

The same pattern applies to any metric that is a non-linear function of the confusion matrix: MCC, Fβ, AUC, calibration ECE. Resample the atomic units (usually item-level predictions), recompute the metric.

For macro-F1 (F1 averaged across classes) with rare classes, the metric can be undefined on a resample where a class disappears; decide up front whether to skip such resamples or return NaN, and report the rule.

## Paired bootstrap for win rates and differences

The most common eval question is a comparison: does Model B beat Model A on this eval? If both models are scored on the *same* items — which is almost always the right way to run the eval — the data are **paired**, and the paired bootstrap is the right tool.

The statistic of interest is a difference:

    Δ̂ = T(x_B) − T(x_A)                   # e.g. accuracy_B − accuracy_A
    or
    Δ̂ = mean over i of 1[B wins on item i] − 1[A wins on item i]   # win-rate margin

The bootstrap resamples *items*, keeping the paired outcomes together:

    for b in 1..B:
        sample indices i_1,...,i_n from {1..n} with replacement
        A_b = per-item outcomes for A on those indices
        B_b = per-item outcomes for B on those indices
        Δ̂_b = T(B_b) − T(A_b)
    CI on Δ̂ = [ percentile(2.5), percentile(97.5) ] over {Δ̂_b}

Two rules that trip people up:

- **Do not resample A independently from B.** If you draw one bootstrap sample for A and a separate one for B, you have thrown away the pairing and the variance of `Δ̂` will be much too large. Same indices for both.
- **The confidence interval is on `Δ̂`, not on the individual estimates.** "The 95% CI on B − A is `[+0.01, +0.05]`" is a meaningful statement. "The 95% CIs on A and B overlap" is not — overlapping marginal CIs can hide a significant paired difference, and non-overlapping marginal CIs can co-exist with a paired CI that includes zero.

Worked mini-example. On `n = 500` prompts, Model B wins 275 times and Model A wins 225 times. Point estimate of B's win-rate is `0.55` (or margin `+0.10`). A paired bootstrap over the 500 paired outcomes yields, say, a 95% CI on the margin of `[+0.02, +0.18]`. That interval excludes 0, so "B beats A on this eval" is a defensible claim at the 5% level. If instead you had computed a Wilson CI on `275/500 = [0.506, 0.593]` and eyeballed whether it overlapped `0.50`, you would get a numerically similar answer here but a systematically different one for smaller effects — because Wilson is the CI on B's rate, not on the margin.

## Ties and the win-rate definition

For pairwise-preference evals, "win rate" is ambiguous under ties. Three common definitions:

- **Strict win rate.** `wins / n`. Ties count as losses. Underestimates the winner if ties are common.
- **Win rate with tie-splitting.** `(wins + 0.5 · ties) / n`. Standard in Elo-style computations.
- **Win rate excluding ties.** `wins / (wins + losses)`. Effective N shrinks; CI widens correspondingly.

Pick one, state it, and use the same convention across the report. The paired bootstrap works with any of them; the outcome vector you resample just changes.

## Common gotchas that make bootstrap CIs lie

- **Clustering ignored.** If your eval has 100 dialogues of 5 turns each (`n = 500` turns), and you bootstrap over turns, the CI is too narrow because turn-level outcomes within a dialogue are correlated. Bootstrap over dialogues (100 resample units) instead; the CI will be wider, and correctly so.
- **Scorer noise ignored.** If the scorer is a stochastic judge, `θ̂` is a random function of the eval set. To account for both sources of noise, run the judge `k` times per item and treat each `(item, run)` pair as an observation, or hierarchically resample items *then* runs.
- **BCa on a boundary-touching statistic without care.** For statistics that saturate at 0 or 1 (accuracy on a small perfect-scoring eval), BCa can fail; percentile is more robust at the cost of accuracy elsewhere.
- **Statistic undefined on a resample.** F1 on a resample with no positive examples; MCC on a resample where a row of the confusion matrix is zero. Pre-decide the handling rule.
- **Reporting "the bootstrap p-value" without stating the null.** Bootstrap gives you a CI; a p-value requires a specific null hypothesis and a test statistic. Chapter 5 covers hypothesis tests directly.
- **Too few resamples.** `B = 100` gives you an unstable 95% CI; the 2.5th and 97.5th percentiles are estimated from only 2–3 order statistics. Use `B ≥ 1000` for a working CI and `B = 10000` for anything you would show a stakeholder.
- **Not seeding.** Bootstrap CIs are reproducible only if the RNG is seeded and the seed is logged. Log the seed with the report.

## When bootstrap is not the right tool

- **The statistic has a closed-form CI that is exact or very accurate.** For a proportion, use Wilson (Chapter 3). Don't bootstrap what you can compute in closed form.
- **The parameter is a tail of the distribution.** Bootstrap CIs on extreme quantiles are unreliable; extreme-value or rank-based methods are more appropriate.
- **You want a CI on a *cross-slice comparison* where the design is unbalanced.** Regression-based methods with clustered standard errors are often more honest.
- **The sample size is tiny (`n < 20`) and the statistic is highly skewed.** The bootstrap has known small-sample failures; consider a permutation test or an exact test.

## Summary

Bootstrap gives you a general CI for any statistic that is a function of the sample: percentile for a single system, paired for a difference between two systems. Resample the atomic unit (item, dialogue, cluster) rather than lower-level observations; use `B ≥ 10000` for reported numbers; log the seed. For a proportion, prefer Wilson; for a comparison, prefer paired bootstrap (or the tests in Chapter 5) over any operation on two marginal intervals.
