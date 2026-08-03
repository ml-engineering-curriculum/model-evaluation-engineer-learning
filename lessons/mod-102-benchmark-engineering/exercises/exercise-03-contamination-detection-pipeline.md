# exercise-03: Contamination Detection Pipeline

**Estimated effort:** 3 hours

## Objective

Build a three-detector contamination pipeline — surface n-gram, semantic embedding, and model-behavioral — and run it against a real public benchmark using a public proxy corpus (and a small model you can query for the behavioral probe). Produce a contamination report in the format Chapter 5 specifies.

The point is to feel the trade-offs each detector makes and see how they *disagree* on real items. A clean report where all three agree is uncommon; the interesting cases are the ones where two detectors flag an item and the third does not.

## Prerequisites

- Chapter 05 of this module.
- Python 3.11+ with `numpy`, `pandas`, `datasets` (Hugging Face), `sentence-transformers` (or another embedding library), `faiss-cpu` (or `hnswlib`), and access to an LLM you can query for log-likelihood — options include `transformers` with a small open-weight model (e.g. a 1B–3B parameter open model whose weights you can load locally) or a hosted API that exposes log-probs.
- Roughly 4–8 GB of disk for the proxy corpus subset.
- Optional but useful: access to `infini-gram` (a public n-gram index over major pretraining corpora) if you have the credentials; otherwise use a local corpus subset as your proxy.

## Target: pick one benchmark and one proxy corpus

Pick one benchmark to attack. All are publicly hosted on Hugging Face `datasets` or on the paper's project page.

- **MMLU** (Hendrycks et al., 2021). Sample 500 items across a handful of subjects.
- **HumanEval** (Chen et al., 2021). All 164 problems.
- **GSM8K** (Cobbe et al., 2021). Sample 500 test-split items.
- **TriviaQA** (Joshi et al., 2017). Sample 500 questions from the unfiltered validation split.
- **HellaSwag** (Zellers et al., 2019). Sample 500 items from the validation split.

Pick one proxy corpus. Match the license and the "large publicly available crawl-derived text" shape:

- **C4** (`allenai/c4`, English subset, license permitting). Sample a manageable subset — a few GB is enough for the exercise.
- **The Pile** (`monology/pile-uncopyrighted`, or the original if licensing allows). Sample similarly.
- **RedPajama v2** (`togethercomputer/RedPajama-Data-V2`). Sample similarly.

The point of a proxy is that the model under evaluation was likely trained on something *derived from* this corpus family, not literally on your snapshot. Any hits you find are a floor on the contamination rate the model actually experienced.

## Model for behavioral probes

Pick one model for which you can obtain per-token log-likelihoods on your benchmark items:

- **A small open-weight model** you can load locally with `transformers`. Any HF checkpoint whose training data cutoff you can look up in the model card. Preferable because you have full log-prob access.
- **A hosted API model** that exposes `logprobs`. Note in the writeup which one and what its training-data cutoff is.

If you cannot run either, do Parts A and B and skip Part C; note the omission in the report.

## Requirements

### Part A — surface n-gram overlap

Write `ngram_detector.py` that, given a set of benchmark items and a proxy corpus, produces a per-item overlap record.

- **Preprocessing.** Lowercase, strip punctuation, collapse whitespace. State the normalization in the report; both under-normalization and over-normalization are defensible, but must be documented.
- **N-gram extraction.** Extract 8-grams (or 13-grams — pick and document one; each has a defensible tradition — GPT-3 used 13-grams).
- **Corpus index.** Build an in-memory set (or a Bloom filter for large corpora) of all n-grams in the proxy corpus. If the proxy is large enough that this is infeasible, sub-sample it and note the sampling rate — the resulting contamination rate is a lower bound.
- **Per-item output.** For each item, compute `ngram_overlap_rate` (fraction of item n-grams found in the corpus), `max_matched_span` (longest matched contiguous span, in tokens), and up to 3 `match_sources` (identifiers of the corpus documents where matches were found).
- **Flag threshold.** Flag an item as `surface_flagged=True` if `ngram_overlap_rate ≥ T` for a T you pick (start with T=0.5 for a strict threshold, T=0.2 for a permissive one). Document your choice.

### Part B — semantic embedding overlap

Write `embedding_detector.py` that runs an embedding-based near-neighbor search.

- **Embedder.** Use a pinned open embedding model (e.g. `sentence-transformers/all-MiniLM-L6-v2` or `BAAI/bge-small-en-v1.5`). Record the model name and version.
- **Corpus index.** Embed all documents (or a manageable sample) of the proxy corpus and index with FAISS or HNSW.
- **Per-item query.** For each benchmark item, retrieve the top-k (k=5) nearest corpus documents. Record `max_cosine_similarity`, and for the top-k, the doc IDs and their cosine scores.
- **Flag threshold.** Flag an item as `semantic_flagged=True` if `max_cosine_similarity ≥ T`. Start with T=0.85 for a strict paraphrase-suspicion threshold. Manually inspect 10 flagged items to sanity-check the threshold and refine.
- **Cross-check with Part A.** Report the overlap between surface-flagged and semantic-flagged items. Discuss any that were surface-flagged but not semantic-flagged (or vice versa).

### Part C — model-behavioral probes

Write `behavioral_detector.py` implementing at least two of the three probes below on your chosen model.

- **Perplexity gap probe.** For your benchmark items and a matched control set (items generated after the model's training cutoff, or items you write yourself in a similar style), compute per-item mean log-likelihood. Compare the distributions (histogram + a paired-bootstrap CI on the mean-per-token log-likelihood difference, mod-101 Chapter 4). A significantly higher likelihood on the benchmark items than the controls is a memorization signal.
- **Quiz-completion probe (Golchin & Surdeanu style).** For 50 benchmark items, prompt the model with the first half of the item (or the item stem for multiple-choice) and check whether it completes to the exact form (question wording, distractor text). Score verbatim-completion rate.
- **Ordered vs. shuffled recall (Oren et al. style).** For a subset with a canonical ordering (e.g. numbered problems), compute the model's likelihood under the canonical order vs. shuffled orders. A significant order-preference is a memorization signature.

Per-item output includes each probe's score and a `behavioral_flagged` boolean per probe.

### Part D — the contamination report

Produce `CONTAMINATION_REPORT.md` following Chapter 5's format. Sections:

1. **Executive summary.** One paragraph: benchmark, proxy corpus, model, detectors run, headline percent-flagged (per detector and any-detector), and a one-sentence recommended action.
2. **Method.** Per detector: algorithm summary, thresholds, corpus versions, tool versions.
3. **Per-item table.** JSONL or CSV, checked into the exercise repo, with columns `item_id`, `surface_flagged`, `ngram_overlap_rate`, `semantic_flagged`, `max_cosine_similarity`, per-probe flags and scores, and `aggregate_severity` (a coded 0–3 severity that reflects how many detectors agree). Do not put this table in the markdown itself; reference it.
4. **Aggregate results.** Overall and per-slice contamination rates. Cross-detector agreement matrix (how often each pair of detectors both flag the same item).
5. **Confidence bounds.** For each detector, describe expected false-positive and false-negative rates and what you did to calibrate (e.g. manual inspection of a sample).
6. **Recommended usage.** For the model under test, which score should be reported — full benchmark, clean subset, or both — and what CI adjustment is warranted (mod-101 Chapter 3–4).
7. **Reproducibility.** Seed, config file references, corpus snapshot versions, and enough shell commands that a reviewer can rerun the pipeline end-to-end.

### Part E — three worked examples

In the report, pick three items and write ~200 words each:

- One item where **all three** detectors agreed on `flagged`. Show the evidence (the matched n-gram span, the near-neighbor corpus doc, the log-likelihood outlier). Argue why this is the strongest form of evidence.
- One item where **surface disagreed with semantic** (surface flagged, semantic did not — or vice versa). Explain the failure mode of the disagreeing detector.
- One item where **behavioral flagged but surface + semantic did not** (or the reverse). Explain what the behavioral probe is catching that the surface/semantic detectors miss.

## Starter guidance

- Do not try to index a full pretraining corpus. A 5–10 GB sample is enough to demonstrate every failure mode; note the sampling rate and treat every flagged rate as a floor.
- Use `datasets.load_dataset(streaming=True)` for the proxy corpus to avoid materializing it all at once.
- For the embedder, batch aggressively — sentence-transformers benchmarks on batches of 32–128 items per GPU call.
- For behavioral probes, the matched control is the hardest part. Post-cutoff items from the same source are ideal; failing that, freshly generated items in the same style are workable but weaker.
- Some benchmarks (HumanEval, GSM8K) have canonical `task_id` orderings; use those for the ordered-recall probe.
- For MMLU, running per-subject gives more informative per-slice numbers than aggregating.
- The manual-inspection step in Part B is important. Pick 10 flagged items and, for each, look at the actual near-neighbor doc. Adjust the cosine threshold based on what you see; do not accept a bare number.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- The pipeline runs end-to-end from a single script or Makefile with a documented seed and produces the per-item CSV/JSONL and the markdown report.
- All three detector modules exist and are used (or the omitted Part C is explicitly and clearly explained).
- Thresholds are stated *and justified* (either by calibration on a manual sample or by citing a prior work's convention). No thresholds pulled from thin air.
- The per-item table has one row per benchmark item and includes every per-detector score and flag.
- The cross-detector agreement matrix is present and discussed.
- The three worked examples in Part E include concrete evidence (matched n-grams, similar-corpus-doc excerpts, log-likelihood numbers), not just the flag booleans.
- The recommended action names one of: report full-benchmark score, report clean-subset score, report both with a note, rotate items, deprecate — and defends the choice.

## Stretch goals

- **Extend to a second benchmark** with the same pipeline and produce a comparative report: which benchmark has higher contamination, on which detectors, against the same proxy corpus.
- **Run against two proxy corpora** (e.g. C4 and The Pile) and produce a per-item table where each item has per-corpus flags. Discuss which detector's decision is most and least sensitive to the corpus choice.
- **Membership-inference at scale.** Use one of the published membership-inference attacks for LLMs (Min-K% probability, ReCall, DetectGPT-style probes) and add it as a fourth detector. Compare against the three baseline detectors.
- **Public-canary check.** If the benchmark ships a public canary string (BIG-bench does), verify whether your proxy corpus contains it. Report the finding — pretraining pipelines that respect canary filters should not include it.
- **Automate the per-slice CI.** Report contamination rates per slice with paired bootstrap CIs (mod-101 Chapter 4) so a downstream reader knows which slices are meaningfully more contaminated than others.
