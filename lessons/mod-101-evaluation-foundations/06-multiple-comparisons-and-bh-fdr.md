# Multiple Comparisons and Benjamini–Hochberg FDR Control

Every eval report worth reading shows *slices* — accuracy by language, by input length, by demographic group, by domain. Every eval report worth reading also shows *multiple metrics* — accuracy, F1, calibration, safety refusal rate. Each of those cells is a hypothesis test, and running many tests in the same report inflates false positives dramatically. This chapter covers the Benjamini–Hochberg (BH) procedure — the false-discovery-rate control you should apply by default — and, more importantly, when you should *not* apply any correction at all.

The correction machinery is well-known. The judgment about when to apply it is where most reports go wrong, so we spend more time on that than on the arithmetic.

## The multiplicity problem in one paragraph

If you run 20 independent tests at α = 0.05 under the global null (no real differences anywhere), the expected number of false positives is 1. If you run 100, it is 5. The eval that reports "significant lift on 3 of 40 slices!" is almost always describing chance. Without a correction, "significant" loses its meaning as soon as the number of tests exceeds a handful.

## Two error rates: FWER vs FDR

There are two ways to control the mess.

- **Family-wise error rate (FWER).** The probability of making *any* false discovery. Bonferroni is the classical example: reject at `α / m` for each of `m` tests. FWER control is strict and appropriate when a single false positive would be very costly — a regulatory claim, a safety gate. It also destroys power fast: at `m = 40`, Bonferroni's per-test threshold is `0.00125`, and small real effects will not survive.
- **False discovery rate (FDR).** The expected fraction of rejections that are false. If you reject 20 tests and control FDR at 5%, on average one of your 20 rejections is a false positive. Weaker than FWER, retains much more power at large `m`, and matches the way eval reports are actually consumed ("of the slices we flagged as regressions, what fraction are real regressions?").

For eval work, FDR is almost always the right target. FWER is right in a few narrow cases (single-shot decisions with high per-error cost); FDR is right for exploratory slice/metric reporting.

## The Benjamini–Hochberg procedure

Benjamini & Hochberg (1995) gave a procedure that controls FDR at a chosen level `q` (e.g. `q = 0.05`) under independence or positive regression dependence of the test statistics:

    1. Compute p-values p_1, ..., p_m from the m tests.
    2. Sort ascending: p_(1) ≤ p_(2) ≤ ... ≤ p_(m).
    3. Find the largest k such that  p_(k) ≤ (k / m) · q.
    4. Reject all hypotheses with p-value ≤ p_(k).

That's it. `scipy.stats.false_discovery_control(pvals, method='bh')` and `statsmodels.stats.multitest.multipletests(pvals, alpha=q, method='fdr_bh')` both implement it.

Two properties:

- **Adaptive threshold.** Unlike Bonferroni's fixed `α / m`, BH's effective threshold depends on how many p-values are small. If there are many strong signals, more tests are rejected; if there are none, BH collapses to something close to Bonferroni.
- **Power at scale.** For `m = 100` tests with 20 true effects, BH will typically reject most of them; Bonferroni will reject almost none. This is why BH is the default in genomics (thousands of tests) and increasingly in ML eval reporting.

If the tests are not independent and not positively dependent (rare in eval work but possible), the **Benjamini–Yekutieli (2001)** variant controls FDR under arbitrary dependence, at the cost of dividing the threshold by `Σ 1/i` for i=1..m (`≈ log(m)`). Use it if you cannot argue for positive dependence; otherwise BH is fine.

## A worked BH example

Suppose you evaluate a new model against a baseline on 10 slices and get the following raw p-values (from paired tests per slice):

    slice     p
    en-US     0.001
    es-ES     0.008
    fr-FR     0.010
    de-DE     0.021
    zh-CN     0.033
    ja-JP     0.045
    ko-KR     0.061
    pt-BR     0.070
    it-IT     0.083
    ar-EG     0.240

Sort ascending (already sorted here). At `q = 0.05` and `m = 10`, the BH thresholds are `(k/10)·0.05 = 0.005, 0.010, 0.015, 0.020, 0.025, 0.030, 0.035, 0.040, 0.045, 0.050`.

Compare each `p_(k)` against `(k/10)·0.05`:

    k    slice     p        (k/10)·0.05   p ≤ threshold?
    1    en-US     0.001    0.005          YES
    2    es-ES     0.008    0.010          YES
    3    fr-FR     0.010    0.015          YES
    4    de-DE     0.021    0.020          NO
    5    zh-CN     0.033    0.025          NO
    6    ja-JP     0.045    0.030          NO
    ...

The largest `k` for which `p_(k) ≤ (k/m)·q` is `k = 3` (`0.010 ≤ 0.015`). BH rejects all hypotheses with p-value ≤ `p_(3) = 0.010`: `en-US, es-ES, fr-FR`.

At Bonferroni with `α = 0.05, m = 10`: reject if `p ≤ 0.005`. Only `en-US`. BH rejects three, Bonferroni rejects one — even at small `m`, BH's adaptive nature buys real power.

## When you MUST correct

- **Reporting slice-by-slice or subgroup-by-subgroup significance across a fixed set of pre-registered slices.** Slice count is the `m`.
- **Reporting per-metric significance across a metric panel** (accuracy, F1, refusal rate, over-refusal rate, factuality, ...) applied to the *same* comparison. Metric count is the `m`.
- **Reporting per-item or per-cluster significance** — "here are the 8 prompts where Model B regressed" from a set of `m = 500` items.
- **Ad-hoc exploratory slicing** ("actually, let me also look at input length quartiles") counts too — the correction has to include the tests you looked at, not only the ones you decided to report. See "garden of forking paths" below.

## When you should NOT correct

This is where most eval reports go wrong in the *opposite* direction — correcting things that should not be corrected, or corrections applied at the wrong level.

- **A single pre-registered primary hypothesis.** If the entire eval was designed to test "does Model B beat Model A on overall accuracy," that is one test, not `m`. Do not divide by the number of other numbers you happened to compute.
- **Different reports of the same underlying quantity.** If accuracy is reported both as a single number and as a breakdown by class, and the breakdown is descriptive rather than hypothesis-testing, only correct the hypothesis-testing family.
- **Numbers reported for descriptive context.** Median latency, throughput, cost per query, output length — these are descriptive statistics, not tests. Do not apply FDR to descriptive summaries; a "CI" on latency is fine, an "adjusted p-value" is a category error.
- **Nested aggregates that share their inputs.** Overall accuracy and per-slice accuracy on the same items are not independent tests — they share data. FDR is not designed for this kind of dependence; report the slice tests as an FDR-controlled family, and report the overall separately.
- **Different evals of different products at different times.** The multiplicity correction is per-family, and a "family" is a coherent set of tests intended to answer a coherent set of questions in a single report. Do not sum across a year of reports.
- **Numbers you did not test.** A CI that does not exclude zero is not an implicit test at level α; you can compute and report a CI without joining the multiplicity family.

The rule of thumb: correct for the tests within a single, pre-declared family that answers a single decision-relevant question. Everything outside that family — descriptive stats, exploratory context, unrelated primary questions — is separately reported and not corrected against the family.

## The garden of forking paths

The multiple-comparison problem has a subtler cousin. Even if you only *report* one p-value, if you *considered* many slicings — "actually, let me try median instead of mean; let me exclude the outliers; let me try log-transformed" — and picked the one that came out significant, the reported p-value is not honest. This is Gelman & Loken's "garden of forking paths."

Two defenses:

- **Pre-register the analysis.** Write down the slicing, the metric, the test, and the α before you look at outcomes. If you deviate, say so and correct for the deviation.
- **Report the whole family, not just the survivors.** If you looked at 40 slices, say so and BH-correct. Reporting only the two that survived without disclosing the other 38 is a form of undisclosed multiplicity.

## Common failure modes in eval reports

- **Reporting an unadjusted "significant at p < 0.05" flag on every cell of a 5-model × 20-slice matrix.** That is 100 tests. On the global null, expect 5 significant cells. Apply BH across the family.
- **Applying Bonferroni to a friendly, positively-dependent slice family and losing all power.** BH is designed for this; reserve Bonferroni for cases where a single false positive is unusually costly.
- **Correcting descriptive statistics.** "Bonferroni-adjusted median latency" is not a thing.
- **Not disclosing the exploratory pass.** If the reported metric panel is the top-5 slices out of an exploratory 40, the reader has no way to compute the honest correction.
- **Applying FDR to a family that includes both accuracy metrics and safety metrics.** These are usually separately-consumed decisions with different acceptable false-positive rates. Split into families and correct each.

## Summary

Report many numbers, correct many numbers. The default correction for eval slice/metric families is Benjamini–Hochberg at some `q` (0.05 for internal decisions, 0.01–0.001 for external claims). Reserve Bonferroni / FWER for the small number of decisions where a single false positive would be very costly. Do not correct descriptive statistics, single pre-registered primary hypotheses, or unrelated reports. The most important habit is transparent disclosure — say what family you tested, what `m` was, what correction you applied, and what you looked at before deciding — because an undisclosed exploratory pass makes any correction meaningless.
