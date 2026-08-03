# Bias, Toxicity, and Fairness at the Model Level

Bias / toxicity / fairness measurement predates jailbreak and dangerous-capability eval by about a decade, and it has more mature instruments — BBQ, StereoSet, BOLD, RealToxicityPrompts, ToxiGen, Perspective API — than any other line item on Chapter 1's dashboard. It is also the line item most likely to be *misreported*, because the tooling produces plausible-looking numbers whose meaning falls apart under a moderate amount of scrutiny: toxicity classifiers with well-documented false-positive patterns on African American English (AAVE) and on disability language, benchmarks whose "stereotype score" saturates on models that are less biased than the benchmark's assumptions, and rater-bias in the human labels that underlie every trained toxicity classifier.

This chapter is about the instruments, the false-positive discipline mod-101 taught applied to them, the rater-bias sources you have to name in a report, and the reporting shape that lets a governance partner distinguish "the model produces less toxic completions on this benchmark" from "the classifier under-flags toxic completions for reasons that are not the model's doing."

## What "bias, toxicity, fairness" mean at the model level

Three overlapping properties, worth distinguishing:

- **Bias** in the ML fairness sense: the model produces systematically different outputs for prompts that differ only in a protected-attribute axis (race, gender, religion, disability status, sexual orientation, nationality, age, ...). "Different" can mean different sentiment, different completion probability, different task performance, or different content. Reference benchmarks: BBQ (Parrish et al. 2022) for stereotype-in-QA, StereoSet (Nadeem et al. 2021) for stereotype-in-completion, WinoBias / WinoGender for occupational stereotypes.
- **Toxicity**: the model produces content that a policy-defined "toxic" classifier flags — slurs, targeted harassment, hate speech, explicit content in contexts where policy forbids it. Reference: RealToxicityPrompts (Gehman et al. 2020) as the standard prompt set, Perspective API and ToxiGen (Hartvigsen et al. 2022) as classifiers.
- **Fairness**: the model's task performance is comparable across demographic groups. In classical ML this is measured by demographic parity, equalized odds, calibration parity, etc. (mod-103 covers the taxonomy). At the LLM level, fairness usually shows up as "does the model answer factual questions about members of group A as well as it does for group B" or "does the model refuse or over-refuse differentially by group" — the mod-103 discipline transferred to a generative setting.

These three overlap but are not synonyms. A model can be biased-but-not-toxic (systematically less helpful to one group without saying anything a classifier would flag), toxic-but-not-biased (generates slurs uniformly regardless of subject), or fair-but-toxic (equally toxic to everyone). Report them separately.

## The two anchor benchmarks and how they can mislead you

### BBQ — Bias Benchmark for QA

BBQ (Parrish et al. 2022) is a multiple-choice bias eval. Each item has a *context* (a sentence or two describing a scene), a *question*, and three answer choices — one for each of two demographic groups involved, plus an "unknown" option. Items come in two forms:

- **Ambiguous** contexts: the context does not disambiguate which subject the question refers to. The correct answer is "unknown." A stereotype-biased model picks the group-consistent-with-stereotype answer.
- **Disambiguated** contexts: the context clearly resolves the referent. The correct answer is now the non-"unknown" option. A stereotype-anti-biased model that over-corrects picks "unknown" here.

BBQ's *bias score* is defined so that a model with zero bias scores 0, a fully-stereotype-biased model scores +100, and a fully-anti-stereotype-biased model scores −100. Accuracy is a separate axis. A well-designed BBQ report shows both:

```
Accuracy (ambig):     0.84  [0.81, 0.87]
Accuracy (disambig):  0.91  [0.89, 0.93]
Bias score (ambig):   +7.1  [+4.3, +10.0]   (stereotype-consistent when ambiguous)
Bias score (disambig): -0.4  [-2.1, +1.3]   (essentially zero when context resolves)
```

The frequent misreport: showing only "BBQ accuracy" and calling it a bias metric. Accuracy on BBQ is (mostly) a general reading-comprehension metric; the *bias* signal is in the delta between how the model answers ambiguous vs disambiguated items and in the direction of the wrong answers. Chapter 2's principle applies: report the vector, not a scalar.

### RealToxicityPrompts + a toxicity classifier

RealToxicityPrompts (Gehman et al. 2020) is 100k prompts drawn from web text, each annotated with the Perspective API toxicity score of the *continuation* found in the original web text. The eval loop is: for each prompt, sample `n` completions from the model, run each completion through a toxicity classifier, report the toxicity distribution.

Two headline metrics from the paper:

- **Expected max toxicity** (EMT): expected value over prompts of the max-over-`n` classifier score on the model's completions. Captures "how likely is the model to *ever* go toxic if you sample it enough."
- **Toxicity probability** (TP): probability that at least one of `n` completions exceeds a threshold (usually 0.5 on Perspective's toxicity axis).

Both are reported per-slice on RealToxicityPrompts' "toxic prompts" split (the prompts whose *original web continuation* was itself toxic — the hardest condition) and on the "non-toxic prompts" split.

The frequent misreport: taking Perspective API's score at face value. Perspective's known failure modes are documented in Sap et al. 2019 ("The Risk of Racial Bias in Hate Speech Detection") and follow-ups: dialectal AAVE is systematically labelled more toxic than syntactically-comparable Mainstream American English, disability language is systematically labelled more toxic than the same content phrased without disability terms, LGBTQ+ identity language triggers false positives at elevated rates. If your model produces more AAVE-flavored completions than the reference (because it was trained on more diverse dialects, say), Perspective will report it as *more toxic* even if a human rater would disagree.

The fix is not to abandon toxicity metrics — it is to name the classifier's known errors, to sanity-check on human-labelled slices, and to always report toxicity results with the classifier version and known-bias caveat.

## The false-positive rate discipline, applied here

Mod-101 Chapter 4 taught false-discovery-rate control across many slices; the same discipline is what turns a plausible bias number into a defensible one.

Two applications:

**Application 1: many-slice reporting.** BBQ has 11 categories (age, disability, gender identity, nationality, physical appearance, race/ethnicity, race x gender, religion, socioeconomic status, sexual orientation, and race x SES). Reporting a "significant bias in nationality" without correcting for multiple comparisons across 11 categories will over-report significance. Use Benjamini–Hochberg (mod-101 Chapter 4) across the category set.

**Application 2: paired-comparison across models.** When comparing model B versus model A on 11 BBQ categories, apply the same FDR discipline to the deltas. A single-model "biased in category X" claim needs its p-value corrected against the other 10 categories tested; a between-model "improved in category X" claim needs correction across all comparisons.

The reporting shape:

```
BBQ (n=58k), post-BH FDR across 11 categories:
  age:                       bias +2.1  [-0.3, +4.5]   q=0.12  (not significant)
  race/ethnicity:            bias +8.9  [+6.2, +11.5]  q=0.001  *
  nationality:               bias +3.2  [+0.8, +5.7]   q=0.04   *
  ...
  * significant at FDR-adjusted q < 0.05
```

Without the correction, you will over-claim category-specific bias findings. With it, the findings that survive are defensible.

## The classifier is the load-bearing component (again)

Every toxicity number in this module inherits the classifier's biases. Two disciplines keep this honest.

### Human-gold calibration on your model's slice

Sample 200–500 model completions stratified by classifier score and by demographic-adjacent content. Have humans (from the mod-106 human-eval discipline) label each as toxic / not toxic under your policy. Compute:

- **Classifier agreement rate with human gold** overall.
- **Classifier false-positive rate on demographic-adjacent content**: fraction of items the classifier flagged as toxic where humans disagreed, sliced by demographic axis of the content (AAVE, disability, LGBTQ+, ...).
- **Classifier false-negative rate on genuinely-toxic content**: fraction of items humans flagged where classifier disagreed.

Report these next to the model's toxicity numbers. The point is not to fix the classifier — you often can't — but to make the reader aware that "toxicity rate 3.1%" is a joint property of the model and the measurement instrument.

### Use multiple classifiers where possible

Perspective API, Detoxify (unitary/detoxify), ToxiGen-trained RoBERTa, and Llama-Guard's toxicity head are the four most-used open toxicity classifiers as of 2026. They disagree substantially on borderline items. A defensible report:

- Runs at least two classifiers.
- Reports the correlation between them.
- Reports the union-of-flags and intersection-of-flags rates.
- Names which classifier's number is the headline and why.

If you have to pick one, pick the classifier whose calibration against human gold is best on *your* model's completions, not the most-cited one from the literature.

## Rater bias in the source data

Every trained toxicity classifier is trained on labels produced by human raters, and the raters are not demographically neutral. The literature has repeatedly shown:

- **Annotator demographics affect labels.** The same content is labelled differently by raters of different racial, cultural, and political backgrounds (Sap et al. 2019, Davani et al. 2022 on annotator disagreement).
- **Annotator identity is not usually captured in released datasets.** Many toxicity datasets do not record the demographic composition of their raters; the resulting classifier inherits an unlabelled and unrepresentative rater distribution.
- **Guidelines drive drift.** Two datasets with the same nominal "toxicity" label but different annotation guidelines produce classifiers with meaningfully different flag rates.

The consequence for your report: **when you cite a toxicity classifier, cite the training-data source and, where available, the rater-population documentation.** If the classifier was trained on data whose raters were 80% one demographic, your toxicity numbers inherit that skew — even if the model's actual output is dialectally diverse, the classifier will produce a demographic-skewed toxicity map.

Where possible, use classifiers that publish rater-demographic-conditioned calibration (a small but growing literature), and prefer classifiers whose training data was labelled by a demographically-representative pool. Where you have to use a classifier without this documentation, name the limitation in your report.

## Fairness at the model level: task-performance disparity

Fairness in the mod-103 sense — comparable task performance across groups — transfers to LLMs as *task performance conditional on demographic content in the prompt or in the reference answer*. Concrete measurements:

- **Refusal / over-refusal by demographic slice.** Take Chapter 2's harmful and benign sets and slice the harmful set by whether the target of the harmful behavior is a specific demographic group; slice the benign set the same way. Report refusal rate and over-refusal rate per slice.
- **Task-performance by demographic slice.** For factual QA benchmarks (TriviaQA, Natural Questions), see whether accuracy varies systematically for questions about members of different demographic groups. Compare with mod-101 CIs and FDR discipline.
- **Sentiment / stance disparity.** For prompts of the shape "Write a bio for a [demographic] doctor" or similar, sample completions and measure sentiment or stereotype-consistency. BOLD (Dhamala et al. 2021) is the canonical benchmark for this; it has known limits and should be one instrument among several.

The reporting shape is the same as for bias: many-slice per demographic axis, FDR-corrected, with CIs and n per slice.

## Contamination and staleness

Two operational issues that recur:

- **Bias benchmarks are widely known.** BBQ, StereoSet, BOLD, RealToxicityPrompts appear in training data corpora scraped in 2022 and later. A model trained after mid-2022 has likely "seen" some of these benchmarks. mod-102's contamination-detection pipeline applies — n-gram overlap against your training corpus, per-item embedding search, canary detection.
- **Slur lists drift.** The vocabulary that a slur classifier flags is a moving target. A classifier trained in 2020 may flag terms that have shifted in usage by 2026 (some reclamations, some new slurs, some archaic terms). Retrain / re-calibrate periodically; report the classifier version.

## A minimum-viable bias/toxicity report

For a model-card safety section (Chapter 7 will pick this up), a defensible mod-109 bias/toxicity subsection includes:

```
Bias (BBQ v1.0, n=58k, all 11 categories, BH-adjusted at q<0.05):
  Categories with significant stereotype-consistent bias in ambiguous items:
    race/ethnicity: bias +8.9 [+6.2, +11.5]
    nationality:    bias +3.2 [+0.8,  +5.7]
    religion:       bias +2.7 [+0.1,  +5.4]
  Disambiguated-item bias: within noise across all categories.
  Overall accuracy (disambig): 0.91.

Toxicity (RealToxicityPrompts, n=100 per prompt, 10k prompts):
  Classifier: Detoxify unbiased v0.5 (calibration on 400 human-gold: agreement 87%,
              known FPR on AAVE per Sap et al. 2019; secondary check with Perspective API v6).
  Expected max toxicity (toxic-prompt split): 0.31 [0.29, 0.33]
  Toxicity probability (threshold 0.5, non-toxic split): 0.024 [0.020, 0.028]
  Per-classifier delta vs Perspective: +0.7 pt on EMT (classifiers disagree slightly).

Fairness (refusal-rate, sliced by demographic target on n=520 harmful prompts):
  Refusal rate per slice, BH-adjusted:
    <table>
  No statistically-significant refusal-rate disparities at q<0.05.

Limitations:
  - BBQ and RealToxicityPrompts appear in some pretraining scrapes; the numbers
    may partly reflect training exposure. Cross-checked with a private held-out
    stereotype set (n=1,200) showing directionally-similar patterns.
  - Toxicity classifiers have documented false-positive patterns on AAVE and
    disability language. Human-gold calibration on 400 items caps the reported
    toxicity rate at ±3 pt confidence.
  - Fairness is measured only on the refusal-rate axis; task-performance
    disparity by demographic remains an open evaluation gap.
```

Three things this deliberately does:

- Reports the *instruments and their limits* alongside the metrics.
- Cites the calibration numbers so the reader knows how much to trust each metric.
- Names the evaluation gap ("fairness on task performance is not measured here") rather than papering over it.

## Guidance for the eval author

- **Report the vector, not the scalar.** BBQ has ambiguous and disambiguated bias, plus accuracy — three numbers per category, not one. Toxicity has EMT and probability at threshold, per split, per classifier — a small table, not one number.
- **Apply FDR across categories.** BBQ, BOLD, and similar benchmarks have many demographic axes; multiple-comparison correction is not optional.
- **Human-calibrate every classifier.** 200–500 model completions labelled by humans, agreement rate reported alongside the metric. If you cannot calibrate, name the classifier's known limitations from the literature.
- **Use at least two classifiers for toxicity.** They disagree; the disagreement is diagnostic.
- **Slice by dialect / demographic where the classifier's known errors live.** AAVE, disability language, LGBTQ+ identity terms. The slice-level false-positive rates are the diagnostic that separates classifier bias from model bias.
- **Report contamination checks.** Every bias benchmark you cite should be paired with a contamination-detection result against your training data (mod-102).
- **Do not confuse bias with toxicity or with fairness.** They are three different measurements. A model can be biased and not toxic; toxic and not biased; fair on task performance and biased in stereotype-completion. Report them separately.

## Summary

Bias, toxicity, and fairness are three related-but-distinct model-level measurements, each with mature instruments (BBQ and StereoSet for bias, RealToxicityPrompts plus a toxicity classifier for toxicity, per-demographic slicing of the Chapter 2 refusal metrics for fairness). The load-bearing skill is *distrust of single-classifier numbers*: toxicity classifiers have well-documented false-positive patterns on AAVE, disability language, and LGBTQ+ identity terms; bias benchmarks saturate and have contamination risk; rater bias in training labels propagates through every classifier. The mod-101 false-positive-rate discipline applies to many-category reporting (Benjamini–Hochberg across BBQ's 11 categories) and to model-versus-model comparisons; the mod-105 judge-calibration discipline applies to every classifier used to score outputs. Report as a vector per benchmark (ambiguous vs disambiguated bias, EMT and probability at threshold per classifier), with human-gold calibration and known-limitations text next to every metric. The next chapter picks up the last measurement on the Chapter 1 dashboard — prompt-injection robustness — and the responsibility line it draws with application-security testing.
