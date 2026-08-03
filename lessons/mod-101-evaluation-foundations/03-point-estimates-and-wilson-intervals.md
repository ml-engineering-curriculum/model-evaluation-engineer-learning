# Point Estimates and Wilson Intervals for Proportions

Most eval metrics eventually reduce to a proportion: accuracy is `correct / total`, refusal rate is `refused / total`, pass@1 is `passed / total`, judge-A-preferred rate is `A_wins / total`. This chapter builds the non-asymptotic confidence interval for a proportion — the Wilson score interval — and shows why it beats the textbook Wald "±1.96·√(p̂(1-p̂)/n)" formula on the sample sizes and score ranges eval reports actually live in.

Chapter 4 covers CIs for more complicated statistics (F1, per-item averages of a bounded score) via the bootstrap. This chapter is the special case where a closed-form interval exists and you should use it.

## The setup

Score `n` items independently. Let `X` be the number of successes. The point estimate of the success probability is

    p̂ = X / n

We want an interval `[L, U]` such that, over repeated experiments with the same `n`, the true `p` is inside `[L, U]` at least, say, 95% of the time. "At least" is a real word here: intervals with exact coverage for discrete data do not exist because the binomial is discrete; the goal is coverage close to the nominal rate without long tails on either side.

## Why the Wald interval is wrong for evals

The Wald interval is the one you see in textbooks:

    p̂ ± z · √( p̂(1 − p̂) / n )

with `z ≈ 1.96` for 95%. It is a normal approximation to the sampling distribution of `p̂`. It is the wrong tool for evals because it fails hardest in the two regimes eval work actually inhabits.

- **Small `n`.** Many eval sets are hundreds, not tens of thousands. HumanEval is 164 problems. GSM8K's canonical test split is 1,319, but common practice reports on subsets. Human-eval Likert studies of 50–200 items are typical. The normal approximation is poor here.
- **Extreme `p̂`.** High-performing models score near 1.0 on saturated benchmarks; safety refusal rates are engineered near 1.0; contamination-probing rates can be near 0.0. Wald near the boundary produces intervals that literally cross 0 or 1. On a 100/100 correct run, Wald reports `[1.0, 1.0]` — a zero-width interval that claims perfect certainty. This is not a rounding artifact; it is a structural failure of the normal approximation near the boundary.

Agresti & Coull (1998) systematically compared several intervals for the binomial proportion and showed that the Wald interval's actual coverage can be far below the nominal 95%, especially at extreme `p̂` and small `n`. They recommended the Wilson score interval (or the closely-related "Agresti–Coull adjusted-Wald") as the default. That recommendation is now standard.

## The Wilson score interval

Edwin Wilson (1927) derived the interval by inverting the score test: `[L, U]` is the set of `p₀` values for which the score-test statistic does not reject `H₀: p = p₀` at level α. Concretely, for confidence level `1 − α` with `z = z_{1−α/2}`:

              p̂ + z²/(2n)          z · √( p̂(1 − p̂)/n + z²/(4n²) )
    center = ─────────────       half-width = ────────────────────────────────
              1 + z²/n                              1 + z²/n

              [L, U] = center ± half-width

Reference implementations you can trust exist in `statsmodels.stats.proportion.proportion_confint(..., method='wilson')` and in `scipy.stats.binomtest(...).proportion_ci(method='wilson')`. Use one of those in production code; the formula is here so you know what they compute.

Properties worth internalizing:

- **The interval stays inside `[0, 1]`.** Even on `X = 0` or `X = n`, both endpoints are strictly interior for any finite `n`. On 100/100 correct at 95%, Wilson reports roughly `[0.963, 1.000]` — a real interval, not a point.
- **The center is shrunk toward 0.5.** The `z²/(2n)` term in the numerator pulls the interval center away from `p̂` toward `0.5`, with the pull vanishing as `n → ∞`. This is the Bayesian intuition: a small sample carries some prior weight toward the middle.
- **Coverage is close to nominal down to small `n`.** Agresti & Coull show coverage stays close to 95% for `n` as small as ~10 and any `p`, whereas Wald's coverage can drop below 85% in the same regime.

## A worked example

Suppose you run a new safety classifier on `n = 200` prompts and it correctly refuses `X = 190`. The point estimate is `p̂ = 0.95`.

Wald 95%:

    0.95 ± 1.96 · √(0.95 · 0.05 / 200)
    = 0.95 ± 1.96 · 0.01541
    = 0.95 ± 0.0302
    → [0.9198, 0.9802]

Wilson 95%:

    center     = (0.95 + 1.96²/400) / (1 + 1.96²/200)
               = (0.95 + 0.00960)   / (1 + 0.01921)
               = 0.9415
    half-width = 1.96 · √(0.95·0.05/200 + 1.96²/(4·200²)) / (1 + 1.96²/200)
               ≈ 0.0311
    → [0.910, 0.973]

The two intervals disagree modestly here; the disagreement grows dramatically as `p̂` approaches 1 or `n` shrinks. Try `n=20, X=20` and you will see Wald report `[1.0, 1.0]` while Wilson reports roughly `[0.839, 1.000]`. That is the difference between a report that claims zero uncertainty and one that admits the sample size can't rule out an 84% true rate.

## What Wilson does not do

- **It does not correct for non-IID sampling.** If the same annotator rated all `n` items, or all `n` prompts came from the same user session, the effective sample size is smaller than `n` and the interval is too narrow. Cluster the resample (Chapter 4 bootstrap) or use a design-based CI.
- **It does not correct for scorer noise.** If the scorer is a stochastic judge, `p̂` is not a fixed function of the eval set; the CI would need to reflect the joint (item, scorer-run) resample. Chapter 4 covers this.
- **It does not correct for eval-set staleness or contamination.** These are validity issues (Chapter 2), not estimation issues. A tight Wilson interval on a contaminated eval is a precise estimate of the wrong thing.
- **It does not give you the CI for a *difference* of proportions.** For paired evals (same items, two systems) use the paired methods in Chapter 5, not two Wilson intervals with overlap-eyeballing. Overlapping Wilson intervals do not imply "no significant difference"; you can have overlapping intervals and a significant paired test, and vice versa.

## The one-line rule

For any eval metric that is a proportion, on any sample size and any point estimate, use the Wilson interval as the default. Wald is a legacy default from a time when computers could not evaluate the Wilson formula; there is no reason to prefer it now.

## Related intervals worth knowing about

- **Clopper–Pearson ("exact") interval.** Guarantees coverage `≥ 1 − α` for every `p`, at the cost of being conservative (real coverage often much higher than nominal, so intervals are wider than they need to be). Use it when the conservative guarantee matters — e.g. safety-critical thresholds where you would rather over-cover than under-cover.
- **Agresti–Coull "adjusted Wald."** Add `z²/2` successes and `z²/2` failures, then run Wald on the adjusted counts. Coverage close to Wilson, formula close to Wald, easy to compute by hand.
- **Jeffreys interval.** Bayesian credible interval with the `Beta(0.5, 0.5)` reference prior. Similar coverage to Wilson; some prefer it because it corresponds to a natural non-informative prior.

For eval work, Wilson is the default; reach for Clopper–Pearson when a report has to make a conservative safety claim, and for Jeffreys if you want the Bayesian framing explicit.

## Summary

Every proportion the eval reports — accuracy, pass rate, refusal rate, win rate — should ship with a confidence interval, and the default that gets it right at eval-typical sample sizes and score ranges is the Wilson score interval. The Wald interval is the wrong tool: it collapses to zero width at the boundary, and its coverage falls off well before the eval sample size gets small. Use `proportion_confint(method='wilson')` or `binomtest(...).proportion_ci(method='wilson')` in code; know the formula so you understand what the library gives you back.
