# exercise-03: RAG Eval with RAGAS and TruLens

**Estimated effort:** 4 hours

## Objective

Stand up a minimal RAG pipeline against a small labelled corpus, evaluate it end-to-end with the four RAGAS metrics *and* the TruLens RAG Triad, calibrate the judge against a 50-item human-labelled slice, and produce a failure-modes writeup that identifies at least three items where a specific metric misled — omission trap, grounded-in-wrong-context, topicality-vs-correctness, or `k`-sensitivity. The deliverable is not a leaderboard-style score; it is a report where the metric numbers are situated inside their known failure modes.

## Prerequisites

- mod-107 Chapters 4 and 5 (RAG metrics; RAG failure modes).
- mod-105 Chapter 5 (calibrating a judge against humans).
- mod-102 Chapter 4 (inter-annotator agreement) — for the labelling step.
- A capable judge model (frontier hosted; do not use the same model as your RAG generator).
- Python environment with `ragas` and `trulens-eval` installed. LangChain or LlamaIndex are optional but simplify the pipeline plumbing.

## The corpus and eval set

Choose a small, self-contained document corpus with public licensing. Good options:

- **Wikipedia subset.** Pick a topic (a single well-scoped subject, e.g. "SQL query language" or "the Renaissance") and export 20–50 articles. Keep the corpus small enough that you can annotate ground-truth thoroughly.
- **A software project's docs.** The React docs, the Django docs, PostgreSQL docs — any project with permissive licensing and a well-defined scope.
- **A published RAG eval set.** MS-MARCO subset, NaturalQuestions subset, or one of the RAGAS-provided examples in the library's docs.

Curate an eval set of **100 items** with:

- `question` — a natural user question the corpus can answer.
- `ground_truth_answer` — a written-out reference answer, 1–4 sentences.
- `ground_truth_context` — the passage id(s) that contain the answer (needed for the classical IR metrics stretch goal and for context-recall interpretation).

The eval set should include, at minimum:

- 20 items whose ground truth is a *single passage*.
- 20 items whose ground truth *synthesizes across two or more passages*.
- 20 items whose ground truth contains a specific *caveat* or negative claim that a cautious generator might drop (omission-trap items).
- 20 items whose question is *ambiguous* — has two plausible interpretations that would retrieve different passages (grounded-in-wrong-context probe items).
- 20 items that are *out-of-scope* for the corpus (a well-behaved RAG system should refuse; a system that answers is fabricating).

Ship `eval_set/items.jsonl` with one JSON object per line.

## Requirements

### Part A — the RAG pipeline

Ship a minimal RAG pipeline in `rag/pipeline.py`:

1. **Chunker.** Split the corpus into 200–500-token chunks with a small overlap. Log the chunker parameters — they matter.
2. **Retriever.** Embedding-based over the chunks. Any embedding model — `text-embedding-3-small`, `bge-small`, `multilingual-e5-large`, whatever fits your setup. Return top-`k` chunks per question.
3. **Generator.** A capable instruction-tuned model, prompted to answer using the provided context. Use a *different* model family than your judge.
4. **Trace logger.** For each item, log `{question, retrieved_contexts, answer, ground_truth_answer, ground_truth_context}` to `logs/traces.jsonl`.

Start with `k = 5`. You will vary `k` in Part C.

### Part B — run RAGAS and TruLens on the traces

**RAGAS pass.** Ship `eval/run_ragas.py`:

- Compute `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall` per item using the RAGAS library. Point the judge at your frontier judge model.
- Aggregate: per-metric mean, per-metric standard deviation, per-item scores.
- Write `logs/ragas.jsonl` with per-item metric values and `REPORT_ragas.md` with the aggregate.

**TruLens pass.** Ship `eval/run_trulens.py`:

- Instrument the pipeline (or replay traces) and compute the RAG Triad — context relevance, groundedness, answer relevance.
- Aggregate similarly.
- Write `logs/trulens.jsonl` and `REPORT_trulens.md`.

Confirm the two libraries produce *directionally consistent* results on the shared metrics (faithfulness/groundedness, answer relevancy/answer relevance, context precision/context relevance). Non-trivial disagreements are a signal — usually a judge prompt difference; note them in the report.

### Part C — the `k` sweep

Rerun the pipeline (or re-invoke the metrics against the same traces if the library allows subsetting) at `k ∈ {1, 3, 5, 10, 20}`. Recompute `context_precision` and `context_recall` at each. Plot the two curves. Comment on the tradeoff. This is the direct measurement of the sensitivity-to-`k` failure mode from Chapter 5.

### Part D — the judge-vs-human calibration slice

Sample 50 items uniformly across the five categories in the eval set. Human-label them yourself (or with a co-labeller) on:

- **Faithfulness (binary).** Is *every* claim in the answer supported by the retrieved context?
- **Coverage / omission (binary).** Does the answer include *every* claim in the ground-truth answer that the retrieved context supports?
- **Relevancy (ternary).** On-topic / partially-off / off-topic.
- **Context recall (binary).** Does the retrieved context contain the information needed to answer?

Compute:

- **Cohen's κ** between human and judge for the two binary metrics.
- **Weighted κ** for the ternary metric.
- **Confusion matrix** per metric.

Report κ alongside the metric aggregates. Below κ = 0.6 on any metric, treat that metric's number as diagnostic-only for this task — the judge is not aligned enough with humans on your task to gate deploys.

### Part E — the failure-modes writeup

The primary deliverable. `FAILURE_MODES.md` (≤ 3 pages) documents at least three items from your eval set where a RAGAS or TruLens metric *lied*. For each:

- **The item.** Question, retrieved context, generated answer, ground-truth answer.
- **The metric reading.** What each of the four RAGAS metrics scored the item.
- **What the metric missed.** Which failure mode from Chapter 5 is at play — omission, grounded-in-wrong-context, topicality-vs-correctness, `k`-sensitivity, judge bias, composite-hides-components — and how you diagnosed it.
- **What a human labeller saw.** The correct verdict on this item from your Part D labelling.
- **What would fix the miss.** Additional metric (coverage), additional slice in the eval set (ambiguity items), better retriever, better generator prompt, better judge, or an explicit "this is not measured by RAGAS" acknowledgment.

The three items should span at least two of the failure modes from Chapter 5. Do not manufacture failures — find real ones in your traces. If your pipeline is too good to produce three failures across 100 items, extend the eval set or weaken the retriever to expose the failure modes; the point of the exercise is the *diagnostic muscle*, not the top-line number.

### Part F — bundle

- `corpus/` — the document corpus (or a manifest with hashes and a fetch script).
- `eval_set/items.jsonl`
- `rag/pipeline.py`, `rag/chunk.py` (or however you split the code)
- `eval/run_ragas.py`, `eval/run_trulens.py`, `eval/human_labels.jsonl`, `eval/calibration.py`
- `logs/traces.jsonl`, `logs/ragas.jsonl`, `logs/trulens.jsonl`
- `plots/k_sweep.png` (from Part C)
- `REPORT_ragas.md`, `REPORT_trulens.md`, `CALIBRATION.md`, `FAILURE_MODES.md`
- `README.md` — reproduction instructions

## Starter guidance

- **Do not use the same model as generator and judge.** Self-preference on faithfulness is the largest single systematic distortion in RAG eval. Use different families (e.g. Claude as judge and Llama-3-Instruct as generator, or vice versa).
- **Do not use the RAGAS defaults blindly.** RAGAS defaults its judge to whatever LangChain LLM you pass in, but the shipped prompts have evolved across library versions. Pin the RAGAS version and the judge model version in the report.
- **The 100-item set is not enough for a headline number, but it is enough to expose failure modes.** Do not try to publish a "RAGAS faithfulness of 0.87" as if it were a leaderboard result. Report it as "0.87 on our 100-item eval, with judge-human κ = 0.72." Wider claims need wider eval sets and are out of scope for a 4-hour exercise.
- **Ambiguity items are the highest-signal category.** They are the ones that expose the grounded-in-wrong-context trap. Spend disproportionate care writing them; ideally use real ambiguity from the corpus (a term that has multiple meanings in the domain), not contrived ambiguity.
- **Out-of-scope items measure refusal appropriateness.** Read the refusal rate as a separate axis. A RAG system that answers 20/20 out-of-scope items is confabulating and will score high on faithfulness (grounded in *something*) and low on correctness — a case study in why composite scores hide the important variable.
- **Label carefully.** Human labels are your ground truth for calibration; a κ of 0.5 vs 0.8 changes the whole conclusion. Read the item, read the retrieved context, read the answer, then label. Do not label from the metric outputs.
- **Faithfulness = 1.0 on a short answer is suspicious.** Look at your short answers manually. They are the most likely to be omission-trap items.
- **Report the metric confusion table.** A single κ is a scalar; the confusion table shows *which direction* the judge disagrees with humans (over-scoring vs. under-scoring). Include it.

## Acceptance criteria

- A working RAG pipeline that produces traces in the four-tensor format (question, contexts, answer, ground_truth) for every item in the eval set.
- Both RAGAS and TruLens produce per-metric aggregates and per-item scores over the eval set.
- The `k` sweep is run and plotted; the plot shows a defensible tradeoff between context precision and context recall as `k` varies.
- The 50-item calibration slice has human labels for all four axes; κ (or weighted κ) is reported per metric.
- `FAILURE_MODES.md` documents at least three real items from your eval set where a metric misled, spanning at least two of the failure modes from Chapter 5.
- The reproducibility manifest pins the judge model, generator model, embedding model, chunker parameters, `k`, library versions, and seeds.

## Stretch goals

- **Passage-level IR metrics.** Use the `ground_truth_context` passage ids to compute classical IR metrics — recall@k, MRR, nDCG — against the retriever. Compare to RAGAS's `context_recall` on the same items. The gap surfaces cases where RAGAS's claim-level definition of recall accepts a "correct enough" passage that a passage-level annotator would flag as a miss.
- **Composite vs. component alerting.** Build a mock alerting rule that fires on the RAGAS composite score dropping below a threshold. Then walk through 5 alerts a component-level rule would have caught earlier. This is the empirical support for "alert on composite, debug on components."
- **Re-ranker impact.** Add a cross-encoder re-ranker (a `sentence-transformers` cross-encoder) between the retriever and the generator. Rerun. Compare context precision @ 5 with and without the re-ranker. This is the intended use of the metric.
- **Judge swap.** Rerun RAGAS with two different judges. Report the per-metric distribution shift and how much of it is model-family vs. judge-prompt drift. This is the most compact demonstration of "judge is a versioned dependency."
- **Adversarial refusal items.** Add 10 items to the eval set whose questions are answerable but only from *contradictory* passages (the corpus has two passages with different facts on the same question). Report what the RAG system does; RAGAS metrics will look plausible but the correctness is undefined. Document the case study.
- **Freshness axis.** If your corpus has timestamped documents, add 10 items whose correct answer depends on the most recent version, and 10 items whose correct answer is from an old version that has been superseded. Report an accuracy split. Freshness is not a RAGAS metric; the exercise here is showing you had to build it separately.
