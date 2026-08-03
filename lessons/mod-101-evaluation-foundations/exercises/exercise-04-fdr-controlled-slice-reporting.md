# exercise-04: FDR-Controlled Slice Reporting

**Estimated effort:** 2 hours

## Objective

Take a supplied (or self-generated) multi-slice eval results table, apply Benjamini–Hochberg FDR control correctly, and produce a corrected report. Along the way, demonstrate that you can distinguish the tests that belong in the family from the numbers that should not be corrected at all.

## Prerequisites

- Chapter 06 of this module.
- `statsmodels` or `scipy` for the BH procedure.

## Requirements

### Part A — get a dataset

Choose one of these paths.

- **Path 1: real public data.** Use a public multi-slice benchmark result table you can point to. Candidates: the HELM "core scenarios" per-scenario table (Liang et al., 2022), the FLORES-200 per-language translation quality table, the BIG-bench per-task table, a HuggingFace `evaluate`-based per-language classification eval. Cite the source. The table needs at least 15 slices and two systems.
- **Path 2: synthetic scenario.** Generate paired outcomes for two "models" across 20 slices with a controllable per-slice sample size. Design the ground truth so that 3 slices have real effects (Δ ∈ {+0.05, +0.08, +0.10}) and 17 have zero effect. Fix a seed and log it.

Either way, land at a table with columns: `slice`, `n_slice`, `system_A_score`, `system_B_score`, and per-slice paired outcomes sufficient to compute a paired test.

### Part B — build the pre-analysis plan

Before you look at p-values, write the pre-analysis plan. This should fit on one page:

1. The **primary hypothesis:** the single decision-relevant question ("does B beat A on the overall metric"). One test.
2. The **slice family:** the set of per-slice tests you will run to characterize *where* the difference lives. Enumerate the slices.
3. The **metric family** (if any): additional metrics (F1, calibration, refusal rate) you will test in addition to the primary. Enumerate them.
4. For each family: the target FDR `q` (default `q = 0.05`), the test to use (McNemar / paired bootstrap / sign test — pick per Chapter 5).
5. Numbers you will report but not correct: latency, throughput, cost, output length. Explain why they are descriptive, not tests.

The plan must exist as a separate file, committed **before** any p-value computation. Reviewer will check the timestamp.

### Part C — run the analysis

Implement the analysis in a notebook or script.

1. Compute the primary test statistic and its CI on the overall metric. Do not correct this against the slice family; it is a separate primary hypothesis.
2. Compute per-slice p-values for the paired test of your choice on each slice in the slice family.
3. Apply BH at `q = 0.05` and report:
    - The full family sorted by ascending raw p-value.
    - The BH threshold `(k/m)·q` for each rank `k`.
    - Which slices survive.
    - The BH-adjusted p-values (`p_i · m / max_rank_i_survives_at`, i.e. the standard BH adjustment).
4. For contrast, also compute the Bonferroni threshold `α/m` and show which slices survive that. Discuss the difference.
5. If your metric family is non-empty, repeat step 3 for the metric family (separately from the slice family — they are not combined).

Present the result as a table with columns: `slice`, `raw_p`, `bh_adjusted_p`, `survives_at_q=0.05`, `survives_bonferroni_at_alpha=0.05`.

### Part D — write the corrected report

Produce a one-page corrected report suitable for a technical audience. It must contain:

- The primary claim ("B beats A on the overall metric") with its CI and p-value. State whether this is significant.
- The slice-family table with BH-adjusted p-values and the survivors listed explicitly.
- A "did not survive" section acknowledging the tests that looked significant at raw `α = 0.05` but did not survive BH.
- A short paragraph on the total family size `m`, whether the pre-analysis plan was followed, and any deviations (with justification).
- A "not corrected" section listing the descriptive statistics reported alongside, with a one-line reason for each ("descriptive, not a test").

## Starter guidance

- The BH computation is a five-line function; do not reach for a library first. Implement it, then verify with `statsmodels.stats.multitest.multipletests(pvals, alpha=q, method='fdr_bh')`.
- BH-adjusted p-values are computed as `p_adj_(k) = min_{j ≥ k} ( m · p_(j) / j )`, then clipped to `[0, 1]`. This is the "enforced monotonicity" step that trips people up on custom implementations. `multipletests` does it for you.
- When picking per-slice tests, be consistent. If most slices are big enough for McNemar `χ²` and a few are small, use McNemar exact everywhere (it agrees with `χ²` at large `b+c` and is more honest at small).
- For the synthetic path, the point of setting 3 true effects and 17 null slices is to give you a small case study of BH's behaviour. Under FDR control at `q = 0.05`, the expected number of false rejections among your rejections is at most `q · (# rejections)`. Sanity-check against this bound.

## Acceptance criteria

Your submission is acceptable if:

- The pre-analysis plan exists as a separate file committed before the analysis (timestamps in `git log`).
- The primary test is reported separately from the slice family and is not subjected to the family correction.
- The BH-adjusted table shows all `m` slices, sorted, with the survivor set explicitly marked.
- Bonferroni is computed for contrast and the report explains why BH was chosen for this application.
- At least one descriptive statistic is reported without a p-value and explicitly labelled "not a test."
- If you chose Path 1, the source is cited with a link; if Path 2, the seed and generating code are committed.

## Stretch goals

- Add a **Benjamini–Yekutieli** (arbitrary-dependence) analysis in addition to BH; compare survivors and discuss whether positive dependence is plausible for your family.
- Simulate the false-discovery rate empirically: for the synthetic path, run the whole pipeline 500 times with different seeds and different true-effect patterns. Verify that the observed false-discovery rate stays below `q`. Plot power vs. `m` at fixed true-effect count.
- Extend the report with an **"exploratory pass"** disclosure — deliberately consider two extra slicings after seeing the data, and demonstrate honest reporting: correct against the larger family that includes them, and show how the correction changes the survivor set.
