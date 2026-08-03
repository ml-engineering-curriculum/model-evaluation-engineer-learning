# exercise-02: Inter-Annotator Agreement Computation and Disagreement Analysis

**Estimated effort:** 3 hours

## Objective

Implement Cohen's κ, weighted κ, Fleiss' κ, and Krippendorff's α from scratch, verify each against a well-tested library baseline, and apply the whole set to a real multi-annotator label file. Produce a disagreement analysis that goes beyond a single point estimate — bootstrap CIs on every reported number, confusion matrices, per-slice κ where slices exist, and a written analysis of the top disagreement modes.

The deliverable is a small library any future exercise can import (`iaa.py`) plus a report on one real label set that a downstream reader could use to decide whether the labels are shippable.

## Prerequisites

- mod-106 Chapter 4 in full.
- exercise-01 complete, OR any real multi-annotator label file with at least 2 annotators, at least 50 items, and at least 3 categorical or ordinal labels. If you do not have your own, use one of:
  - The label CSV from `exercise-01/data/pilot_labels.csv`.
  - The public Chatbot Arena human-preference dataset (`lmsys/lmsys-arena-human-preference-55k` on Hugging Face — pairwise votes on prompt / response-A / response-B triples).
  - Any of the Hugging Face datasets tagged with multi-annotator labels (`ucberkeley-dlab/measuring-hate-speech`, for example).
- Python with `numpy`, `scipy`, `sklearn`, `statsmodels`, and the `krippendorff` package.

## Requirements

### Part A — `iaa.py` from scratch

Implement the following functions with `numpy`-only bodies (no library agreement statistics inside your implementations — you compare against libraries in Part B).

```python
def cohen_kappa(labels_a, labels_b, categories=None) -> float:
    """Cohen's κ for two raters on categorical labels."""

def weighted_cohen_kappa(
    labels_a, labels_b, categories, weights="quadratic"
) -> float:
    """Weighted Cohen's κ; weights in {'linear', 'quadratic'}."""

def fleiss_kappa(counts_matrix) -> float:
    """Fleiss' κ. counts_matrix[i][j] = # raters who labelled item i as
    category j. Rows must sum to the same N (fixed raters per item)."""

def krippendorff_alpha(
    reliability_data, level="nominal"
) -> float:
    """Krippendorff's α on a (raters x items) matrix. NaN entries mean
    the rater did not label that item. level in {'nominal', 'ordinal'}."""

def bootstrap_ci(
    stat_fn, args, n_resamples=2000, ci=0.95, seed=0
) -> tuple[float, float, float]:
    """Return (point, lo, hi) via nonparametric bootstrap over item index.
    stat_fn takes the same signature as the four statistics above."""
```

Guidance:

- For **Cohen's κ**, marginal `p_e` is `sum(p_a[c] * p_b[c] for c in categories)`.
- For **weighted κ**, generalize by weighting each (i, j) cell of the confusion matrix and the outer product of the marginals with the same weight matrix; `κ_w = 1 - sum(w * observed) / sum(w * expected)`.
- For **Fleiss' κ**, per-item agreement is `(sum(n_ij² - N) / (N * (N-1)))` where `n_ij` is the count of raters assigning item i to category j and N is the raters-per-item; overall `P̄` is the mean over items; `P̄_e = sum(p_j²)` where `p_j` is the pooled marginal.
- For **Krippendorff's α**, implement the standard `1 - D_o / D_e` form with the appropriate distance function per level: nominal is `d(c, c') = 0 if c == c' else 1`; ordinal is `d(c, c') = (c - c')²` after mapping labels to integer ranks. Handle NaN cells by only counting pairs where both raters labelled the item.
- For the bootstrap, resample *item indices* with replacement, not rater indices. The unit of independence is the item.

### Part B — validation against libraries

Ship `test_iaa.py` that verifies your implementations against library baselines on at least three synthetic and one real case each:

- `cohen_kappa` vs. `sklearn.metrics.cohen_kappa_score`.
- `weighted_cohen_kappa` (linear and quadratic) vs. `sklearn.metrics.cohen_kappa_score(..., weights=...)`.
- `fleiss_kappa` vs. `statsmodels.stats.inter_rater.fleiss_kappa` (with `aggregate_raters` to build the counts matrix).
- `krippendorff_alpha` vs. `krippendorff.alpha(..., level_of_measurement=...)`.

Match to within `1e-9` on the synthetic cases (they should be numerically identical). If they don't match, your implementation has a bug — fix it before proceeding to Part C. Do not "adjust" the library output to match yours.

Sanity check: on a two-rater complete-data nominal case, `cohen_kappa` and `krippendorff_alpha(level='nominal')` should match to within a small correction factor (Krippendorff's α uses `N-1` normalization; Cohen's κ uses `N`). Verify this and document the small difference.

### Part C — apply to a real dataset

Pick one of the datasets from Prerequisites. Ship `analysis/report.py` that produces `analysis/report.md` covering:

1. **Dataset summary.** Item count, number of annotators, label shape (categorical vs. ordinal), items-per-annotator distribution.
2. **Marginal label distributions.** Per-annotator, plus overall pooled. This is the diagnostic that tells you whether κ will be prevalence-sensitive on this data.
3. **Primary agreement statistic.** Pick the right one from Chapter 4's decision table for the data's shape. Point estimate + bootstrap 95% CI (2,000+ resamples).
4. **A cross-check statistic.** If you reported Cohen's κ or Fleiss' κ, report Krippendorff's α as well. If the two disagree by more than ~0.05, the report should analyze why (typically prevalence sensitivity or missing-data handling).
5. **Confusion matrix.** For two-rater cases, the full matrix. For multi-rater cases, pairwise matrices for at least three annotator pairs. This is the diagnostic; every agreement number is uninterpretable without it.
6. **Per-slice κ.** If the dataset has meaningful slices (topic, length bucket, difficulty), report κ per slice with ≥ 20 items. Report the max-min per-slice κ delta; a large delta means the aggregate κ is hiding heterogeneity.
7. **Top disagreement modes.** Read at least 10 items where annotators disagreed most. For each of the top 3–5 patterns you see, write a one-paragraph description: which annotators, which labels, what the item looks like, what the likely underlying reason is.
8. **Shippability call.** Given the point estimate, the CI, the confusion matrix, and the top disagreement modes, is this label set shippable for (a) leaderboard / preference ranking, (b) training-signal use, (c) safety-critical downstream use? Answer per use case, not as a single number.

### Part D — bundle

Ship in the exercise directory:

- `iaa.py`
- `test_iaa.py`
- `analysis/report.py`
- `analysis/report.md`
- `data/` — the input label file (either a symlink to exercise-01's output, a small vendored copy, or a download script if the source is public and permissively licensed)
- `run.sh` — one-shot pipeline: run tests, produce the report

## Starter guidance

- **Implement one statistic completely and verify it before starting the next.** Cohen's κ is the easiest to verify. Get it matching sklearn to 1e-9 before touching weighted κ. Then use the weighted κ implementation with `w_ij = 1{i≠j}` as a Cohen's κ cross-check.
- **The bootstrap resamples items, not raters.** Resampling raters is a different (and rarely wanted) statistic — it treats raters as the population you're sampling from.
- **NaN handling in Krippendorff's α is the failure mode.** Test it explicitly: build a case where rater 3 skipped items 5 and 10, verify your α matches the library's α with the same NaN pattern.
- **Prevalence sensitivity is easy to demonstrate.** Build a synthetic 100-item binary label set that is 95% positive, with the two raters agreeing 92% of the time. Report κ. Then rebalance to 50% positive with the same 92% observed agreement (you'll need to design the confusion matrix carefully). κ will be very different at the same observed agreement. This is worth demonstrating in your report.
- **Do not treat "the library disagrees with me" as evidence the library is wrong.** In every case where students report a library bug in `sklearn.metrics.cohen_kappa_score` or `statsmodels.stats.inter_rater.fleiss_kappa`, the bug has been in the student's implementation. Debug your implementation first.
- **On the real dataset, do not skip Part C.7.** The confusion matrix and the top disagreement modes are what turn a κ number into actionable information. A report that ships a κ = 0.62 (0.55–0.69) point estimate with no confusion matrix and no prose is not usable by anyone downstream.

## Acceptance criteria

- All four statistics in `iaa.py` are implemented with numpy-only bodies; no imports of library agreement statistics inside the implementations themselves.
- `test_iaa.py` verifies each implementation against a library baseline on at least three synthetic cases + one real case each; synthetic cases match to `1e-9` on nominal statistics.
- The real-dataset report includes: dataset summary, marginals, primary statistic with bootstrap 95% CI (2,000+ resamples), a cross-check statistic, at least one confusion matrix, per-slice κ if slices exist with ≥ 20 items, and a prose analysis of at least 3 disagreement modes.
- The shippability call is per-use-case (leaderboard / training signal / safety-critical), not a single number.
- The bundle is reproducible: `bash run.sh` runs the tests, produces the report, and exits nonzero if any implementation test fails.

## Stretch goals

- **Weighted-κ vs. unweighted-κ gap.** On an ordinal task, compute both weighted and unweighted κ. Write the paragraph interpreting the gap: adjacent-category disagreements produce a large gap (weighted is generous; unweighted is punishing correctly); distant-category disagreements produce a small gap.
- **Prevalence-corrected agreement.** Implement PABAK (Prevalence-Adjusted Bias-Adjusted Kappa; Byrt et al. 1993) as an alternative to κ on skewed data. Compare κ, α, and PABAK on the same skewed synthetic case and discuss which is most defensible.
- **Chance model choice.** Cohen's κ uses each annotator's own marginal distribution as the chance model; Scott's π and Krippendorff's α use the pooled marginal. Implement Scott's π and demonstrate the (small) difference on your real dataset. Include a note on when the choice matters (large annotator-marginal differences).
- **Agreement over time.** If your real dataset has timestamps, split the annotator sessions into halves and compute κ on each half. Rising κ over time is annotator training; falling κ is drift or fatigue. Both are useful signals.
- **Interactive confusion-matrix explorer.** A tiny Jupyter notebook or Streamlit app that lets a reviewer click a cell in the confusion matrix and see the items in it. Reviewers who can see the items find failure modes an aggregate κ hides.
