# Reading Eval Reports Critically

The previous six chapters give you the tools. This chapter gives you the reading protocol — a repeatable checklist for taking someone else's eval report and locating the validity, sampling, statistical-power, and contamination weaknesses. It is the skill you will use most often on the job, because you will read many more eval reports than you will author.

## The four questions to ask of any report

For any headline claim ("Model B beats Model A on our internal benchmark by 3.4 points"), work through these four in order:

1. **Validity.** Does the metric measure the construct in the claim? (Chapter 2)
2. **Sampling.** Where do the items come from, and how well does the sampling frame match the deployment surface? (Chapter 2, external validity)
3. **Statistical power.** Is the reported difference distinguishable from noise at the sample size and paired-agreement structure of the eval? (Chapters 3–5)
4. **Contamination.** Is any part of the eval set in the training data of any of the systems being compared? (Chapter 2 construct validity; mod-102 for methods)

Order matters. A statistically-significant result on a contaminated benchmark is precisely-estimated memorization. A well-powered slice comparison on a non-representative slice generalizes to nothing. Fix validity and sampling before you interrogate the numbers.

## The 12-item red-flag checklist

For each item, "yes" is fine; "no" is a red flag that goes in your review.

**Setup**

1. Does the report state what construct the metric operationalizes, and acknowledge the gap? (E.g. "we use LLM-judge preference as a proxy for user preference; calibration to a human panel shown in appendix.")
2. Is the eval set specified by version and hash, with a link or a citation?
3. Are the model, prompt template, decoding parameters, and seed specified for every system compared?
4. Is the scorer specified — including judge model, judge prompt version, and judge decoding — for every scored run?

**Comparison**

5. Are systems compared on the *same items* (paired), and is the comparison reported as a paired statistic rather than two marginal numbers?
6. Does every reported difference have a confidence interval, and is the CI method disclosed (Wilson for a proportion; percentile or BCa bootstrap otherwise)?
7. Is the sample size large enough to detect the claimed effect? Is a minimum-detectable-effect stated?
8. If binary paired outcomes, is McNemar (exact if small `b+c`) reported? If preference outcomes, is the sign test or paired-preference bootstrap reported?

**Slices and multiplicity**

9. If per-slice or per-metric results are reported, is a multiple-comparison correction applied (BH at some `q`)?
10. Is the total number of tests in the family disclosed — including exploratory slices that did not make the headline table?

**Validity guardrails**

11. Is a contamination check reported — item-level lookup, n-gram overlap, or canary-set performance? For a public benchmark, is the result contextualized against the plausibility of pretraining exposure?
12. Is the eval set representative of the deployment surface, and is any known mismatch flagged?

You will rarely get 12/12 on a real report. Two or three "no"s are typical for a well-written internal report; more than five is usually enough to send it back for revision.

## Reading protocol: a worked walkthrough

Consider this fictional-but-representative claim from a fictional-but-representative model-card update:

> *"On our v3.2 helpfulness benchmark (n=500 prompts), Model B outperforms Model A with a 62% pairwise win rate as judged by GPT-4o, up from 51% for the previous release. Per-language slices show B ahead in 7 of 10 languages. Recommendation: promote B."*

Work the four questions:

### 1. Validity

- Construct: the claim is about "helpfulness," but the operationalization is "GPT-4o preference." The report should either (a) show calibration of the judge against a human helpfulness panel — Cohen's kappa or Spearman correlation — or (b) hedge the construct claim explicitly. Absent (a), treat the number as "GPT-4o-preferred rate," not "helpfulness."
- Construct-irrelevant variance: three likely channels: (i) length — is B systematically longer or shorter than A, and does the judge reward length? (ii) position bias — was the judge run swap-and-averaged, or was A always listed first? (iii) self-preference — GPT-4o judging GPT-4o-family outputs is a known bias if either model shares lineage with the judge.
- Rubric: was the judge given a written rubric with anchors, or a bare "which is more helpful" prompt? Bare prompts absorb the judge's own idiosyncratic definition of "helpful."

### 2. Sampling

- Where did the 500 prompts come from? User traffic sampled last quarter? A hand-curated "helpfulness suite"? User-submitted from an early-access channel? The distribution shift between the sampling frame and the deployment surface is the first-order external validity question.
- Language distribution: if 7-of-10 languages show B ahead, what are the item counts *per language*? If most languages have `n = 20`, most slice comparisons are underpowered and the "7 of 10" pattern is nearly what you would get from noise (see the multiplicity item below).

### 3. Statistical power

- 62% of 500 is 310 wins to 190 losses (or with ties, some split). A paired-preference CI on `p̂ = 0.62, n = 500` (Wilson approximation, though McNemar / sign-test is stricter) is roughly `[0.577, 0.661]`. The 62% is meaningfully greater than 50%. Good.
- The previous release was 51%. The prior "51 vs 50" claim, at `n = 500`, has a Wilson CI around `[0.466, 0.554]` — a null result. The claimed *improvement from 51 to 62* is a comparison of *two win rates against A*, and needs its own CI; do not read it as two independent decisions.
- Per-language slices: with 500 items across 10 languages, the median language has ~50 items. From the MDE table in Chapter 5, a `b + c = 50` paired binary test at 80% power detects only effects of ~0.20 above 0.5. Any slice-level "B ahead" reading needs a slice-specific CI, not just a directional flag.

### 4. Contamination and multiplicity

- Contamination: helpfulness-preference evals typically use novel or in-house prompts, so contamination against training data is less of a concern than for a factual benchmark. But: is any prompt in the eval set drawn from publicly available assistant-eval corpora (Anthropic HHH, LMSYS Chatbot Arena logs, etc.) that Model B or the judge may have trained on? Ask.
- Multiplicity: "7 of 10 languages" is 10 tests. Under a global null of 50/50 in each language, the probability of at least 7 being nominally "ahead" (unadjusted, binomial(10, 0.5)) is around 17%. That is not extraordinary. BH-adjust the 10 slice p-values and report how many survive.

### Bottom line for this report

The overall 62% claim, if the judge is honestly calibrated and the prompt sample is representative, is a real effect. The "7 of 10 languages" claim is essentially uninformative at the reported sample sizes and needs slice-level CIs plus multiplicity correction. The recommendation to promote B is defensible on the aggregate metric but the per-language subclaim should not appear in the model card without stronger analysis.

## Reading protocol for a benchmark-leaderboard claim

The problem shape is different when the eval is a public benchmark (MMLU, GSM8K, HumanEval, GPQA). The four questions specialize:

- **Validity → contamination first.** Frontier-model releases post-2023 should be assumed to have exposure to almost any pre-2023 benchmark unless a decontamination story is documented. A "state-of-the-art on MMLU" claim from 2026 without a decontamination check is not evidence about capability; it is evidence about training-corpus curation. See mod-102 for detection.
- **Validity → prompt-format sensitivity.** Small changes in chat template, system prompt, or answer-extraction regex move MMLU-style multiple-choice scores several points. If the report does not fix and publish the exact prompt, the comparison across systems is not apples-to-apples.
- **Sampling → task decomposition.** MMLU is a mixture of 57 subject-area subsets with wildly different item counts and characters. An overall score is a weighted average; the choice of weighting (macro vs. micro; equal-subject vs. equal-item) changes the ranking. Ask which averaging was used.
- **Power → per-subject counts.** For a per-subject or per-task comparison, sample sizes are small (some MMLU subjects are under 100 items). Per-subject win claims need slice-level CIs and BH correction.
- **Comparison → same code.** Was the challenger evaluated with the same harness (lm-evaluation-harness version, HELM commit, Inspect version) as the baselines it is compared against? Reharnessing a baseline with a different scorer or prompt template quietly shifts the ranking.

## Reading protocol for a production A/B claim

Mod-110 covers this in depth. The 30-second version:

- Was there a pre-registered primary metric and null hypothesis?
- Were guardrail metrics pre-registered, and did any violate?
- Is the CI on the primary metric wider than the reported effect, or narrower?
- Was CUPED (or similar variance reduction) used, and is the variance-reduction ratio disclosed?
- If the analysis is sequential or continuously-monitored, is a sequential-testing correction applied (α-spending, always-valid confidence sequences)?

Every one of those questions maps onto one of the four in our master protocol; the machinery is the same.

## A short glossary of tells

Phrases that should raise your alertness:

- **"Beats the baseline"** without a CI on the difference → statistical conclusion validity concern; ask for the paired CI.
- **"Significant at p < 0.05"** with `n < 100` and no multiplicity correction → underpowered plus multiplicity concern; ask for the family size and the MDE.
- **"State-of-the-art on [public benchmark]"** without decontamination discussion → construct validity via contamination.
- **"Judge preferred"** without judge calibration → construct validity via judge bias.
- **"Human-eval"** with `n < 50` and no IAA → statistical + rubric validity concerns.
- **"Consistent improvement across all N slices"** at small per-slice sizes → underpowered slice claims presented as consistent when they are consistent with noise.
- **"Improved by X.Y%"** where X.Y is smaller than the eval's known noise floor → statistical conclusion validity.

None of these tells are proof of a bad report; they are cues to slow down and ask specific questions.

## The 60-second review

If you have only a minute, ask three things:

1. **What is the CI on the headline number?** If it isn't reported, that is the review.
2. **What is the family, and was it corrected?** For any per-slice or per-metric result.
3. **What is the contamination story?** For any comparison on a public benchmark.

That is 80% of the value of a longer review. The rest of the checklist matters when the answers to those three raise no red flags and you are deciding whether to promote a model.

## Summary

Reading an eval report is a protocol, not a vibe. Work the four questions — validity, sampling, power, contamination — in order; use the 12-item checklist to keep score; watch for the small set of tells that reliably indicate a shallow analysis. Almost every real report will fail some items; the reviewer's job is to identify which failures are load-bearing for the specific decision the report is trying to inform and which are cosmetic. The exercises in this module (particularly exercise-05) drill this protocol on realistic report samples.
