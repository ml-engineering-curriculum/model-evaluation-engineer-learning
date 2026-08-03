# Paired Comparison Tests: McNemar, Paired Bootstrap, Sign Test

Chapter 4 gave you a paired-bootstrap CI on the difference between two systems. Sometimes the decision you have to make is binary — "is B better than A, yes or no, at the 5% level" — and a hypothesis test is the more natural framing. This chapter covers the three tests you will use most often, when each is appropriate, and why *power* — not p-value size — is the thing you should actually be tracking at eval-typical sample sizes.

## When paired tests, and when unpaired?

If both systems are scored on the *same* items — the default in every well-designed eval — the observations are paired and the paired tests are strictly more powerful than the unpaired alternatives (independent-sample t-test, two-proportion z-test). Use the paired form.

Unpaired tests apply when systems are scored on *different* items, e.g. because you A/B'd them on live traffic and each request went to one arm. Different problem, different chapter (mod-110 covers CUPED, sequential testing, and the online A/B toolkit); this chapter is offline paired eval.

## The three tests, in one paragraph each

**McNemar's test.** For paired binary outcomes (each system either correct or incorrect on each item). Test statistic is a function of the *discordant* pairs — items where the two systems disagreed. Under `H₀: P(A right, B wrong) = P(A wrong, B right)`, the discordant count has a known distribution, and McNemar's `χ²` is the classical approximation. Dietterich (1998, "Approximate statistical tests for comparing supervised classification learning algorithms," *Neural Computation*) is the canonical reference in ML and explicitly recommends McNemar for paired classifier comparison.

**Paired bootstrap test.** For any statistic — accuracy, F1, ROUGE, mean Likert score, win rate. Resample paired items with replacement, recompute `Δ̂`, and use the resulting distribution to compute either a CI or a p-value against the null of no difference. The mechanics are Chapter 4; the framing is a hypothesis test. Use this when the metric is not a simple proportion.

**Sign test.** For paired outcomes where all you can observe is which of the two systems was better on each item (a preference or a strict ordering), with no reliable magnitude. Under `H₀: P(B > A) = 0.5`, the number of "B > A" items is `Binomial(n, 0.5)` (excluding ties). The p-value is the two-sided binomial tail. Distribution-free, requires almost no assumptions, and is the natural test for pairwise-preference data from an LLM judge or a human panel.

## Picking the right one

    ┌─────────────────────────────────────────────────────────────────────┐
    │  What do you observe per item?                                      │
    ├─────────────────────────────────────────────────────────────────────┤
    │  Both A and B correct/incorrect                     → McNemar       │
    │  A per-item score (F1, ROUGE, Likert) for each      → paired boot   │
    │  Just "B better," "A better," "tie" per item        → sign test     │
    └─────────────────────────────────────────────────────────────────────┘

Two secondary considerations if more than one applies:

- **When the metric is a proportion and n is small.** Prefer McNemar. It uses the exact structure of the binary paired data and is more powerful than a bootstrap treating each 0/1 outcome as a continuous score.
- **When the metric is a proportion but you also want a CI on the *difference in proportions*.** Compute both — McNemar for the test, paired bootstrap for the CI. They agree on significance and give you the interval as a bonus.

## McNemar's test in detail

Set up the 2×2 contingency table of paired outcomes:

                    B correct    B wrong
    A correct         a            b
    A wrong           c            d

Only `b` and `c` — the "discordant" cells — carry information about the difference. The classical `χ²` McNemar statistic is

    χ² = (b − c)² / (b + c)     ~   χ²₁    under H₀

with continuity correction `(|b − c| − 1)² / (b + c)` in some implementations.

The approximation is unreliable when `b + c` is small (rule of thumb: `b + c < 25`). For small discordant counts, use the **exact McNemar test**, which computes the binomial p-value directly:

    p = 2 · P( Binomial(b + c, 0.5) ≥ max(b, c) )

Or the **mid-p variant**, which is less conservative and better-calibrated than the exact test for discrete data.

`statsmodels.stats.contingency_tables.mcnemar(table, exact=True, correction=True)` gives you all three variants; default to `exact=True` unless `b + c` is comfortably large.

Two things to notice about McNemar:

- **The effective sample size is `b + c`, not `n`.** If two models agree on 480 of 500 items and disagree on only 20, you have a test with `n_eff = 20`. That is a small test regardless of how many total items you evaluated. See the power section below.
- **McNemar tests marginal homogeneity, not accuracy equality per se.** The null is that the *rate of A-only-right disagreements* equals the *rate of B-only-right disagreements*. For paired binary eval that is the same as "same accuracy," but if the paired outcomes were something other than correct/incorrect (say, category assignments), you would be testing something more specific.

## Paired bootstrap as a hypothesis test

Two equivalent ways to derive a test from the paired-bootstrap machinery of Chapter 4:

- **CI inversion.** Compute the bootstrap CI on `Δ̂`. Reject `H₀: Δ = 0` at level α if `0` is outside the `(1 − α)` CI. This is the easy way and it is what most people mean when they say "the bootstrap said it's significant."
- **Recentered bootstrap p-value.** Compute the bootstrap distribution of `Δ̂ − Δ̂_obs` — the deviation of the bootstrap statistic from the observed value — and take the two-sided p-value as `p = 2 · min( P(Δ̂*_b − Δ̂_obs ≥ 0), P(Δ̂*_b − Δ̂_obs ≤ 0) )` (Efron & Tibshirani, Chapter 16). More accurate for small samples and near the boundary.

The paired-bootstrap test's power depends on the metric: it can be much more powerful than McNemar when the metric is continuous (per-item scores) and the two systems differ by small amounts on many items rather than by large amounts on a few. Do not use it as a substitute for McNemar on binary outcomes; use it when the metric is not binary.

## Sign test

For pairwise-preference data — a judge (LLM or human) picks "B better," "A better," or "tie" on each item — the sign test asks: among the non-tie items, does one side win more than half the time?

    let m = number of non-tie items
    let k = number of B wins
    p = 2 · P( Binomial(m, 0.5) ≥ max(k, m - k) )       # two-sided

Equivalent to McNemar's exact test in the two-alternative case; the naming comes from a different literature but the arithmetic is the same. Use the sign-test framing when your data is *natively* preference-shaped (no correctness ground truth, only a pairwise winner) — LLM-judge Arena comparisons, human side-by-side studies.

Ties. Standard practice is to drop tie items and compute the test on the remaining `m` non-ties. If ties are informative (e.g. "both refused," "both got it right"), the sign test loses information; consider the more general Wilcoxon signed-rank test or a bootstrap on a tie-splitting win-rate.

Weakness. The sign test uses only the direction of each item, not the magnitude — it treats a barely-preferred B and a decisively-preferred B identically. When magnitudes are meaningful and available, a paired bootstrap on a scored outcome is more powerful.

## Power: why you should worry about it more than the p-value

Statistical **power** is the probability of detecting a true effect of a specified size, at a specified significance level. Formally, `1 − β = P( reject H₀ | H₁ true )`. Underpowered evals produce false negatives: real improvements that the test cannot distinguish from noise. Chasing a small p-value on an underpowered eval is a category error; you should first ask whether the test is powered to detect the effect you care about.

Two facts to internalize:

- **The effective sample size for paired binary tests is the discordant count, not `n`.** If A and B agree 90% of the time on a 500-item eval, `b + c ≈ 50`. To detect a true 60/40 split among discordant pairs at 80% power and 5% significance requires roughly `b + c ≥ 200` (from the exact binomial power formula) — four times what you have. On this eval, at this agreement rate, you cannot reliably distinguish moderate paired effects.
- **The effective sample size for pairwise-preference tests is the non-tie count.** If the judge ties 70% of the time, an "n=500" preference study has only ~150 informative items. At that size, detecting a true 55/45 preference at 80% power takes roughly `m ≥ 780` non-ties — far beyond 150.

### A minimum-sample sanity check

Before you run a paired eval, do the power calculation. For a paired binary comparison at α = 0.05, two-sided, 80% power, minimum detectable effect (MDE) in the discordant probability:

| Discordant `b + c` | MDE on `P(B right \| discordant)` above 0.5 |
|---:|---:|
| 50 | ± 0.20 |
| 100 | ± 0.14 |
| 200 | ± 0.10 |
| 500 | ± 0.063 |
| 1000 | ± 0.044 |

(Normal approximation: MDE ≈ `(z_{α/2} + z_β) · 0.5 / √(b+c)` = `1.401 / √(b+c)` at α=0.05 two-sided, 80% power. Off by a few percent from the exact binomial power calculation in the small-sample regime; reproduce with `statsmodels.stats.proportion.samplesize_confint_proportion` or an exact binomial power routine for reported numbers.)

Read the row that matches your expected discordant count. If your target real-world effect is smaller than the MDE, you are not powered to detect it — collect more items, or expect null results even when there is a real difference.

### A one-line rule for pre-registration

Every paired eval should ship with a pre-registered minimum-detectable-effect statement: "at n=X and expected disagreement rate d, this eval is powered to detect win-rate differences of size δ or larger at 80% power / 5% significance." If you cannot write that sentence honestly, run more items before you look at the numbers.

## What p-values are and are not

- A p-value is `P( data as extreme or more extreme | H₀ true )`. It is *not* the probability the null is true, and it is *not* the probability the observed effect is real.
- A p-value of 0.04 on an underpowered test is weak evidence. A p-value of 0.06 on a highly powered test is meaningful evidence *against* a large effect. Read p-values and power together.
- Statistical significance is not effect size. Report the effect size (paired-bootstrap CI on `Δ`, McNemar's `b/(b+c) − 0.5`, sign-test win rate) alongside any p-value. A significant 0.2-point improvement on a benchmark whose noise floor is 3 points is not a shippable improvement.
- A single test is a single decision. Multiple tests over slices or metrics create a multiplicity problem; see Chapter 6.

## Summary

Three tests cover the vast majority of paired eval comparisons: McNemar for paired binary outcomes, paired bootstrap for continuous per-item scores, sign test for native preference data. All three are more powerful than any unpaired alternative and should be the default when both systems saw the same items. The single most common failure mode at eval scale is not test choice — it is running an underpowered eval and interpreting a null result as evidence of "no difference." Power-check before you run; report the minimum detectable effect with the result.
