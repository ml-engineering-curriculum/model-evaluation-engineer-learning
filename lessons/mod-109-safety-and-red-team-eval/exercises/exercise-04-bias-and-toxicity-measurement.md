# exercise-04: Bias and Toxicity Measurement With FPR and Rater-Bias Discipline

**Estimated effort:** 4 hours

## Objective

Run a bias / toxicity / fairness measurement against a candidate model that satisfies the Chapter 5 reporting shape: BBQ with ambiguous-and-disambiguated bias scores per category, RealToxicityPrompts with at least two toxicity classifiers, a demographic-slice fairness check on refusal rates, and — critically — a false-positive-rate audit that names the classifiers' known error patterns on AAVE, disability, and LGBTQ+ language. The deliverable is a report a governance partner could consume that clearly distinguishes "the model is more toxic on X slice" from "the classifier is more false-positive-prone on X slice." Every category-level comparison is Benjamini–Hochberg-corrected.

## Prerequisites

- mod-109 Chapter 5 (bias / toxicity / fairness).
- mod-101 Chapter 2 (bootstrap CIs) and Chapter 4 (FDR).
- mod-105 (judge bias controls).
- Exercise-01 for the refusal-rate slice reuse.
- Python 3.11+, `datasets`, `numpy`, `scipy` (for BH).
- Access to at least two toxicity classifiers: Detoxify (`unitary/detoxify`), Perspective API (Google), and/or ToxiGen-trained RoBERTa. At least two are required.

## Datasets

Load from pinned revisions:

- **BBQ** (Parrish et al. 2022), from `heegyu/bbq` or the official repo. All 11 categories. Note the ambiguous / disambiguated split structure.
- **RealToxicityPrompts** (Gehman et al. 2020), from `allenai/real-toxicity-prompts`. Random subsample of 5,000 prompts if the full 100k is too expensive; report `n` explicitly. Split into `toxic-prompt` and `non-toxic-prompt` subsets per the dataset's `challenging` field.
- **Optional: BOLD** (Dhamala et al. 2021) for stereotype-in-completion, from the official Amazon Science release. Use if time permits.

For the fairness-slice check, reuse the harmful-request set from exercise-01 with an added `demographic_target` label (per HarmBench's item metadata where available; otherwise annotate a subset).

## Requirements

### Part A — BBQ measurement

Build `bias_eval/bbq.py`:

- Sample the model on every BBQ item (multiple-choice) at `temperature=0`. Log the model's chosen answer.
- Compute per category, for both ambiguous and disambiguated items:
  - **Accuracy** with 95% bootstrap CI (2000 resamples, seed=17).
  - **Bias score** per the BBQ paper's formula: signed score in `[-100, +100]` measuring stereotype-consistency.
- Apply Benjamini–Hochberg correction across the 11 categories on the ambiguous-item bias scores' non-zero hypothesis. Report `q` per category.

### Part B — toxicity measurement

Build `bias_eval/toxicity.py`:

- For each RealToxicityPrompts item, sample `n=25` completions from the model at temperature 1.0 (per the paper's protocol; or `n=10` if compute-limited, note in report).
- Score each completion with at least two classifiers (Detoxify + Perspective API or Detoxify + ToxiGen classifier). Store `(prompt_id, completion_index, classifier_a_score, classifier_b_score)`.
- Compute per split (toxic-prompt, non-toxic-prompt), per classifier:
  - **Expected max toxicity** (EMT): expected value over prompts of `max_i score_i`.
  - **Toxicity probability** at threshold 0.5: probability at least one of `n` completions exceeds threshold.
- Report both, plus per-classifier disagreement rate (fraction of `(prompt, completion)` pairs where the two classifiers give opposite verdicts around threshold 0.5).

### Part C — classifier calibration and false-positive audit

Build `bias_eval/calibration.py`:

- Sample 400 model completions stratified by:
  - Classifier score (100 each in `[0, 0.25]`, `[0.25, 0.5]`, `[0.5, 0.75]`, `[0.75, 1.0]`).
  - Content axis: 100 items whose prompts or completions contain AAVE-associated language, 100 disability-related terms, 100 LGBTQ+ identity language, 100 no-flag "neutral" content. Use a public dialect / demographic lexicon or a simple keyword list from published sources (cite the source).
- Hand-label (yourself, plus one reviewer if possible) each as toxic / not-toxic under a written policy you draft (at least a page).
- Report:
  - Overall classifier agreement with human gold.
  - Per-slice false-positive rate (classifier said toxic, human said no).
  - Per-slice false-negative rate.
  - Slice-vs-neutral FPR ratio (a well-known Perspective failure mode is elevated FPR on AAVE — quantify it on your model's outputs).

The calibration report is the load-bearing artifact for the section: without it, your toxicity numbers are hostage to the classifier's biases and cannot be interpreted.

### Part D — fairness slice (refusal-rate disparity)

Build `bias_eval/fairness.py`:

- Reuse the harmful-request set from exercise-01. Label each item with a `demographic_target` field (which demographic group is the target of the harmful behavior). Use HarmBench's own metadata where available; where not, hand-label a subset.
- Compute refusal rate per demographic group with 95% CI.
- Apply Benjamini–Hochberg across the demographic axis. Report `q` per group.

### Part E — the report

```
# Bias, toxicity, and fairness report

Model:         <provider/model@version>
Datasets:
  - BBQ:                     revision <sha>, all 11 categories, n = 58k
  - RealToxicityPrompts:     revision <sha>, n_prompts = 5000, n_completions/prompt = 25
  - Fairness slice:          harmful set from exercise-01, n = 300

Classifiers:
  - Detoxify unitary/detoxify v<version>   (agreement with human gold: <A>%)
  - Perspective API v<version>              (agreement with human gold: <B>%)

## Bias (BBQ)
| Category (ambiguous) |   n   |  Accuracy [CI]    | Bias score [CI]       |  q   | Sig |
|----------------------|-------|-------------------|-----------------------|------|-----|
| age                  | 3680  | 0.84 [0.83, 0.85] | +2.1 [-0.3, +4.5]     | 0.12 |     |
| race/ethnicity       | 6412  | 0.79 [0.78, 0.80] | +8.9 [+6.2, +11.5]    | 0.001| *   |
| ...                  |       |                   |                       |      |     |

Disambiguated bias (all categories): within noise; overall accuracy 0.91.

## Toxicity
| Split         | Classifier     | EMT [CI]           | P(tox>0.5) [CI]     |
|---------------|----------------|--------------------|---------------------|
| toxic-prompt  | Detoxify       | 0.31 [0.29, 0.33]  | 0.42 [0.39, 0.45]   |
| toxic-prompt  | Perspective    | 0.33 [0.31, 0.35]  | 0.44 [0.41, 0.47]   |
| non-toxic     | Detoxify       | 0.04 [0.03, 0.05]  | 0.024 [0.020, 0.028]|
| non-toxic     | Perspective    | 0.05 [0.04, 0.06]  | 0.028 [0.024, 0.032]|

Classifier disagreement (Detoxify vs Perspective, threshold 0.5): 6.2%.

## Classifier calibration and FPR audit
Overall agreement with human gold (n=400):
  Detoxify:    87%
  Perspective: 84%

Per-slice false-positive rate:
| Slice            |  n   | Detoxify FPR | Perspective FPR | FPR vs neutral (Detoxify) |
|------------------|------|--------------|-----------------|---------------------------|
| AAVE-associated  | 100  |  0.18        |  0.24           | 3.6x                      |
| Disability terms | 100  |  0.14        |  0.19           | 2.8x                      |
| LGBTQ+ identity  | 100  |  0.12        |  0.17           | 2.4x                      |
| Neutral          | 100  |  0.05        |  0.07           | 1.0x  (baseline)          |

Reading: reported EMTs on demographic-adjacent slices are subject to
classifier FPR inflation of 2.4x-3.6x relative to neutral content.
Any per-slice toxicity finding on those slices must be treated as
classifier-noise-first, model-behavior-second.

## Fairness (refusal-rate disparity)
| Demographic target | n  | Refusal rate | 95% CI       | q    | Sig |
|--------------------|----|--------------|--------------|------|-----|
| race group A       | 42 | 0.95         | [0.85, 1.00] | 0.42 |     |
| race group B       | 38 | 0.92         | [0.81, 0.98] | 0.42 |     |
| ...                |    |              |              |      |     |

No statistically significant refusal-rate disparities at q<0.05.

## Limitations
- BBQ appears in some public pretraining corpora; contamination is a known
  possibility. Private held-out audit is a stretch goal.
- Toxicity classifiers used inherit Sap et al. 2019-documented dialectal
  false-positive patterns; the calibration audit quantifies the inflation
  on this model's outputs at 2.4x-3.6x.
- Fairness is measured only on refusal-rate axis; task-performance fairness
  by demographic is an unfilled evaluation gap.
- Non-English coverage is not evaluated in this run.
```

## Starter guidance

- **Start with the calibration audit.** It anchors every downstream number. If you skip it, your headline toxicity rate is a classifier-noise-first, model-second number, and a governance partner will see through the omission.
- **Do not conflate bias, toxicity, and fairness.** They are three separate measurements. Each has its own section, its own instruments, its own limitations.
- **Apply FDR everywhere you slice.** BBQ has 11 categories; toxicity has 2 splits × 2 classifiers; fairness has N demographic groups. Every table with multiple hypotheses gets a `q` column.
- **Report the vector on BBQ.** Ambiguous and disambiguated, accuracy and bias score, per category. Do not compress. The paper's own reporting shape is the reference.
- **Cite the calibration source.** For your dialect lexicon or demographic-term list, cite the published source. Do not make it up.
- **Content sensitivity.** Toxicity classifier outputs are numbers, not content; keep completions in a controlled store. The calibration audit's per-slice sample is inevitably going to touch difficult content — treat those raw samples with the same discipline as the other exercises' controlled stores.
- **`n` and CI everywhere.** Every rate carries its `n` and its CI. If you slice small (some BBQ subcategories have `n<50`), the CI is wide — the report must reflect that.

## Acceptance criteria

- BBQ reported per category with both accuracy and bias score, ambiguous and disambiguated, BH-adjusted at `q<0.05`.
- RealToxicityPrompts reported with EMT and P(tox>0.5) per split, per classifier (at least two classifiers), plus classifier disagreement rate.
- Classifier calibration audit exists: `n≥400`, per-slice false-positive rates on at least AAVE, disability, and LGBTQ+ slices, with the FPR-vs-neutral ratio reported.
- Fairness slice reports refusal rate per demographic target with BH correction.
- Limitations section names contamination risk, classifier FPR patterns, and evaluation gaps.
- No raw completions in the report; per-item logs live in a controlled store with a documented retention policy.
- The report reproduces from a `python -m bias_eval.report` CLI with pinned seeds and dataset revisions.

## Stretch goals

- **BOLD.** Add BOLD for stereotype-in-completion, sliced per demographic axis, with sentiment or classifier-based scoring. Report next to BBQ as a triangulating instrument.
- **Rater-demographic-conditioned calibration.** If you can arrange a small human-gold labelling team with recorded demographic diversity, re-run the calibration on the same 400 items with the diverse team and report the label disagreement between the diverse team and a homogeneous baseline. This is the Sap et al. / Davani et al. rater-bias signal, quantified.
- **Multilingual toxicity.** Extend to at least one non-English language on RealToxicityPrompts-multilingual (or Jigsaw multilingual). Report per-language and note the classifier-support caveat.
- **Private held-out BBQ-like set.** Author, under bias-policy review, a small (n=200) private held-out bias set that does not overlap BBQ. Compare bias scores. A gap is the contamination signal.
- **Task-performance-fairness pilot.** Take a factual-QA benchmark (TriviaQA subset) and slice it by whether the question is about members of different demographic groups. Report accuracy per group with FDR. Task-performance fairness is a well-known coverage gap and this is a durable first pass.
- **Judge-rationale content audit.** For every classifier "toxic" flag that survives calibration, sample 50 items and audit that the classifier's underlying content is *actually* what a governance policy would consider toxic. This is the "is the classifier measuring the right thing?" audit that is easy to skip and often surfaces further FPR patterns.
