# Detecting Benchmark Contamination

Contamination is when items from your benchmark — or content close enough to them to matter — appeared in a model's pretraining or fine-tuning data. When they did, the model's score on those items is a mix of capability and memorization, and the number stops meaning what it says. mod-101 Chapter 2 called this a construct-validity failure; this chapter is the operational side: how to detect it, how to produce a report that quantifies it, and what to do when the detector fires.

Three families of detectors are covered — surface (n-gram / substring), semantic (embedding overlap), and model-behavioral (log-likelihood and membership-inference probes) — because each catches a failure mode the others miss. A serious contamination report runs all three.

## Why contamination is now the dominant construct-validity threat for LLM benchmarks

For public benchmarks (MMLU, HumanEval, GSM8K, TriviaQA, HellaSwag, BIG-bench) that predate a model's training cutoff and were posted somewhere on the crawlable web, treat presence in pretraining as the default assumption. Sainz et al. (2023) surveyed the state of NLP benchmark contamination and argued that per-benchmark contamination reports are now table stakes for any published eval; Golchin & Surdeanu (2023) proposed model-behavioral probes for the case where the training corpus is not available. Since 2023 the standard has continued to tighten: model cards for major releases increasingly include a contamination section, and eval consumers are learning to distrust reports that don't have one.

The eval engineer's job is not to eliminate contamination — that requires access to the training corpus, which you often do not have. It is to *quantify* it, to isolate the affected items, and to give the reader enough information to decide whether the remaining uncontaminated portion of the benchmark is large enough to trust.

## What counts as contamination

The strict definition is: the benchmark item, or a substring/paraphrase of it, appeared in the training corpus. The operational definition breaks it into several severities:

- **Verbatim item leakage.** The full item — question and possibly answer — appears in pretraining. Score on this item is memorization. Highest severity.
- **Answer-only leakage.** The reference answer appears near the question in pretraining (e.g. an answer key on the same page). Model can memorize the answer keyed by the question. High severity.
- **Test-item paraphrase.** A close paraphrase or translation of the item appears. Model may transfer memorization to the eval-form item. Medium-high; depends on how paraphrase-invariant the task is.
- **Test-set structure leakage.** The item is novel but its format, distractors, or scaffolding are copied. Model has learned the test format. Low-medium; matters more for multiple-choice than for open-ended generation.
- **Ancillary leakage.** Solutions, discussion, or spoilers to the item exist online (competitive-programming problems with published editorials; standardized test items with study-guide walkthroughs). The model may have memorized the reasoning trace even if not the item verbatim. Medium.

A contamination report names which severities it detected and at what rate; it does not conflate them into a single "contaminated: yes/no" bit.

## Surface detection: n-gram and substring overlap

The simplest, oldest detector — used in essentially every serious LLM release paper — is an n-gram overlap check between the benchmark and the pretraining (or a proxy) corpus.

### The basic algorithm

For each benchmark item, extract the input text (and optionally the reference answer). Compute n-grams (usually 8-, 13-, or 50-token contiguous spans). Query the pretraining corpus for exact matches. Flag items whose n-gram overlap exceeds a threshold.

Choices to make explicitly:

- **n-gram length.** Longer n-grams have exponentially lower false-positive rates but catch fewer paraphrases. 8-grams are the informal default for surface-form matching; the GPT-3 report used 13-grams; some newer work uses 50-grams for near-verbatim only.
- **Normalization.** Lowercase, strip punctuation, collapse whitespace. Aggressive normalization catches more real matches but inflates false positives; underdo it and you miss trivial variations.
- **Match threshold.** Fraction of item n-grams found in the corpus. GPT-3 used ≥ 70% n-gram overlap as its filter for flagged items. Whatever you choose, publish it.
- **Direction.** Item-in-corpus is the usual check; corpus-in-item catches the rarer case where the item embeds a widely-quoted passage.

### Implementation shortcuts

The naive implementation ("for each n-gram, search the corpus") is intractable at pretraining scale. The tractable approaches:

- **Bloom filters over corpus n-grams.** Constant-time lookup, small false-positive rate you can control. Standard in production contamination pipelines.
- **Suffix arrays or FM-index over the corpus.** Exact matching in sublinear time per query. Used by, e.g., ROOTS and Pile analyses.
- **`infini-gram` and similar open-source indices.** Precomputed n-gram indices over public pretraining corpora (C4, The Pile, RedPajama). Query-able without hosting the full corpus.

If you do not have access to the model's actual pretraining corpus (the usual situation for closed-weight models), pick a proxy: the union of the largest public pretraining corpora is a reasonable lower bound on what the model has seen. The overlap you measure is a floor, not a ceiling.

### The report artifact

Per benchmark item, record:

- **`ngram_overlap_rate`** — fraction of item n-grams found in the (proxy) corpus.
- **`max_matched_span`** — longest matched contiguous span, in tokens.
- **`match_sources`** — up to `k` corpus URLs or document IDs where matches were found.
- **`contamination_severity`** — a coded severity per the taxonomy above.
- **`corpus_version`** — the version of the corpus queried.

The aggregated report has, at minimum: per-slice contamination rate, distribution of overlap rates, and a "clean" vs. "flagged" per-item flag with the threshold used.

## Semantic detection: embedding overlap

Surface matching misses paraphrases, translations, and re-narrations. Embedding-based detection catches them.

### The algorithm

Embed every benchmark item and every candidate corpus document into a shared vector space (a sentence encoder, or an LLM's mean-pooled last hidden state). For each item, find the corpus documents with cosine similarity above a threshold. Manually or heuristically classify each near-neighbor as "same item," "paraphrase," "topically similar but distinct," or "unrelated."

Choices:

- **Embedder.** A well-benchmarked sentence encoder (Sentence-BERT variants, `text-embedding-3-small`/`-large`, `bge`, `E5`). Pin the version — embeddings from different models are not comparable.
- **Similarity threshold.** Not universal; calibrate on a held-out sample where you know the ground-truth match/non-match status. Typical starting points: cosine ≥ 0.90 for near-duplicate / paraphrase suspicion; ≥ 0.75 for topical relevance.
- **Corpus index.** FAISS or a similar approximate-nearest-neighbor library. Approximate is fine; missing 1% of the neighbors barely moves the aggregate contamination rate.

### What embedding detection catches that n-gram does not

- **Paraphrases and rewordings.** An item asking "What is the capital of France?" and a corpus document saying "The capital city of France is Paris" have zero n-gram overlap and high embedding similarity.
- **Translated items.** Cross-lingual embeddings catch translation contamination that surface matching misses entirely.
- **Restatement in a different format.** Multiple-choice items reformatted as free-response, or vice versa.

### What it misses

- **False positives from topical similarity.** Two independent items on "the capital of France" are semantically identical without the second being contamination.
- **Adversarial paraphrases that shift semantics slightly.** Small semantic changes reduce cosine similarity below the threshold while the model still exploits the memorization.
- **Contamination via reasoning chains.** The item is novel; the model has memorized a similar problem's solution.

Use it alongside n-gram, not instead of.

## Model-behavioral detection: log-likelihood probes

When the pretraining corpus is unavailable, the only signal you have is the model itself. Model-behavioral detectors ask the model questions whose answers are indicative of memorization vs. genuine capability.

### Perplexity-gap probes

Under a well-known intuition: for text the model has memorized, the model's per-token log-likelihood on the item is higher (perplexity lower) than on structurally equivalent text it has not seen. Concrete procedures:

- **Members vs. non-members.** For a candidate contaminated set (e.g. the public HumanEval problems) and a matched non-member control set (e.g. HumanEval-style problems generated after the model's cutoff), compare the distribution of per-token log-likelihoods. A significantly higher mean on the members is evidence of contamination.
- **Item-level rank.** For each benchmark item, compute its log-likelihood under the model. Rank items by likelihood. Items in the top quantile are the memorization suspects. Combined with the surface / semantic evidence, they harden the story.

### Structural probes

- **Ordered vs. shuffled recall (Oren et al. 2023, "Proving Test Set Contamination").** For a dataset with a canonical ordering (e.g. numbered problems), test whether the model's likelihood on the canonical order is systematically higher than on shuffled orderings. Under the null of no contamination, order should not matter; a strong ordering signal is a memorization signature.
- **Quiz-completion probes (Golchin & Surdeanu 2023, "Time Travel in LLMs").** Prompt the model with a partial benchmark item and see whether it completes it to the exact form (question stem, distractor text) rather than a plausible variation. Verbatim completion of item boilerplate is a memorization signal.
- **Multiple-choice option shuffling.** If reshuffling the options in a multiple-choice item drops accuracy sharply, the model has memorized the option-to-answer mapping, not the reasoning.

### What the probes cannot tell you

- **Exactly which items were memorized.** Behavioral probes give a per-item probability of contamination, not a proof.
- **The corpus of origin.** You know memorization happened, not where from.
- **Whether the memorization is decision-relevant.** A model that memorized every item and still scores 60% has not made the score meaningful, but the fact that it did not saturate tells you something. Report the probe distribution alongside the surface / semantic evidence, not instead of it.

Recent work has emphasized that behavioral membership-inference has weak power at LLM scale on average, but the *combination* of ordered-recall + quiz-completion + surface-overlap gives a much sharper picture than any of them alone. Publish all three.

## The contamination report

A per-benchmark contamination report has, at minimum:

- **Executive summary.** One paragraph naming the corpora queried, the detectors run, and the headline "% items flagged as likely contaminated" with a definition of "flagged."
- **Method.** For each detector: exact algorithm, thresholds, corpus versions, tools and their versions.
- **Per-item table.** `item_id`, per-detector flags and scores, aggregate severity. Publish the full table with the benchmark artifact; it is joinable to per-item scores for downstream re-analysis.
- **Aggregate results.** Overall contamination rate, per-slice contamination rate, cross-detector agreement (how often the three detectors agree on an item), example flagged items with their evidence.
- **Confidence bounds.** For each detector, a discussion of false-positive and false-negative rates on your calibration sample.
- **Recommended usage.** Which score to report — full-benchmark, uncontaminated-subset, or both. If the uncontaminated subset is small, say so and adjust the CI accordingly (mod-101 Chapter 3/4).
- **Reproducibility.** Enough detail (seeds, config files, container images) that a third party can regenerate the report.

The report is a first-class artifact of the benchmark release, not an appendix. Chapter 6 hashes it into the manifest.

## What to do when the detector fires

Options, in order of what you should reach for first:

1. **Report the contaminated subset separately.** The clean, uncontaminated subset gets the headline number; the contaminated subset gets a second number, clearly labelled. Consumers who care about "is this benchmark still meaningful" get their answer.
2. **Rotate to fresh items.** Retire the contaminated items, replace with new ones sourced after the model's cutoff (Chapter 1's `retrieved_at` field is what you filter on). This is Chapter 6's "minor rotation" flow.
3. **Deprecate the benchmark.** If contamination is pervasive and rotating would require replacing most items, the benchmark has reached end-of-life. Deprecate cleanly and point to the successor. Chapter 6 covers the policy.
4. **Add canaries.** In parallel, add or expand a canary set (Chapter 7) so that the *next* generation of models can be measured for post-release inclusion.

What not to do:

- **Do not silently drop flagged items.** Drops must be logged in the release notes and reflected in the manifest.
- **Do not re-run only on the clean subset without publishing the flagged results.** A reader deciding whether to trust the number needs to see both.
- **Do not treat a passing detector run as absolution.** The detector's power is not 100%; a clean report is evidence, not proof.

## Cost and cadence

Contamination detection is not free. Rough shape:

- **Surface n-gram, one-time, against public corpora.** Hours to a day of compute per benchmark against `infini-gram`-style indices. Redo on every corpus update.
- **Semantic embedding, one-time.** Embedding cost dominates; a few dollars per thousand items on hosted APIs, or a few GPU-hours self-hosted, per benchmark.
- **Behavioral probes, per model.** Cost scales with model API pricing and item count. A few hundred to a few thousand items × the model's per-token cost.

Standing cadence:

- **On benchmark release.** All three families, full report, published.
- **On each model release you evaluate.** Behavioral probes against the specific model. Surface / semantic checks can be reused unless the corpus changed.
- **Quarterly, on stable benchmarks.** Re-run surface / semantic against updated corpora and republish the report.

## Summary

Benchmark contamination is a first-order construct-validity threat for public LLM benchmarks and is best treated as present-until-proven-otherwise. Detect with three complementary families — n-gram / substring overlap against the pretraining or a proxy corpus, embedding overlap for paraphrases and translations, and model-behavioral probes (perplexity gaps, ordered recall, quiz completion) for cases where the corpus is unavailable. Publish a first-class contamination report with per-item flags, per-detector scores, thresholds, and reproducibility metadata. When the detectors fire, report the clean and contaminated subsets separately, rotate items, or deprecate. Chapter 6 shows how the contamination report participates in the benchmark version and how rotations become minor version bumps.
