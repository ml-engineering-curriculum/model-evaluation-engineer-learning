# RAG Metrics with RAGAS and TruLens

Retrieval-augmented generation stops fitting a single-turn eval the moment you look at it seriously. The system under test is a *pipeline*: a retriever pulls `k` passages from a corpus, and a generator writes an answer conditioned on those passages plus the user's question. The user sees only the final answer. But the eval has to peer inside the pipeline, because a wrong answer can fail in three different places — the retriever missed the right passage, the retriever pulled the right passage but the generator ignored it, or the generator invented content the passages did not support — and the fix depends on which one it is.

RAGAS (Es et al. 2023) and TruLens (TruEra, 2023) are the two libraries the community converged on for this. Both instrument the pipeline with a small family of *reference-free* (and, optionally, reference-full) metrics computed by an LLM judge over the pipeline's intermediate state. The metric names are stable across both — *faithfulness*, *answer relevancy*, *context precision*, *context recall* — and each name maps to a specific question about a specific pipeline slot. This chapter walks each metric, what it is actually asking the judge, what a good value looks like, and what its inputs are. The next chapter walks the failure modes.

## The RAG pipeline as instrumentation surface

A minimal RAG loop:

```
question ──▶ retriever ──▶ retrieved_contexts (list of passages)
                                    │
                                    ▼
                              generator ──▶ answer
```

The eval-instrumentation surface is the three named tensors: `question`, `retrieved_contexts`, `answer`, plus optionally `ground_truth_answer` and `ground_truth_context`. RAGAS calls these `question`, `contexts`, `answer`, `ground_truth`. TruLens exposes them as feedback-function inputs. Every metric in this chapter is a function of some subset of these five things.

The `ground_truth_context` (which passages *should* have been retrieved) and `ground_truth_answer` (what the answer *should* have said) are the things a labelled RAG eval set has. Reference-free metrics work without them; reference-full metrics require them. Most production teams operate with a small (100–500 item) labelled evaluation set they curate carefully, plus a large unlabelled traffic sample they run the reference-free metrics over. Both are useful; neither replaces the other.

## The four RAGAS metrics

### 1. Faithfulness

**Question the metric answers.** "Is the answer *grounded in* the retrieved context? Does every claim in the answer trace back to a passage the retriever returned?"

**Inputs.** `question`, `answer`, `contexts`.

**How RAGAS computes it.** Two-step LLM judge chain:

1. Extract the atomic claims (statements) from the answer. E.g. from "Paris is the capital of France, and it has 2.1 million residents," extract [`Paris is the capital of France`, `Paris has 2.1 million residents`].
2. For each claim, ask the judge whether it is *supported by* (entailed by) the concatenated retrieved context.
3. Score = (fraction of claims that are supported).

**What a good value looks like.** 1.0 means every claim in the answer is grounded. 0.0 means none are. On a well-tuned RAG system with strong retrieval and a compliant generator, faithfulness > 0.9 is realistic; below 0.7 usually indicates the generator is drifting off the context (hallucinating or synthesizing from its priors).

**What it does not tell you.** Whether the answer is *complete*, whether it addresses the question, or whether the context was itself correct. It is a *self-consistency of pipeline* metric, not a correctness metric. See Chapter 5 for the failure modes.

### 2. Answer relevancy

**Question the metric answers.** "Does the answer address the question, or does it wander?"

**Inputs.** `question`, `answer`. (No context needed.)

**How RAGAS computes it.** Reverse-generation trick:

1. Ask a judge model to *generate `n` candidate questions* that the given answer would be a good answer to.
2. Compute the embedding-similarity between each generated candidate question and the original question.
3. Score = mean cosine similarity.

The intuition: if the answer actually addresses the original question, then the questions the answer would be a good response to should be close (in embedding space) to the original question. If the answer is off-topic, the generated questions will drift.

**What a good value looks like.** 0.9+ on well-formed answers; low values indicate rambling, evasive, or off-topic replies.

**What it does not tell you.** Whether the answer is *correct*. A confidently wrong answer that stays on topic scores high. Answer relevancy is a *style/topicality* proxy, not a truthfulness proxy.

### 3. Context precision

**Question the metric answers.** "Of the passages the retriever returned, which ones actually contain information relevant to the question — and are the relevant ones ranked at the top?"

**Inputs.** `question`, `contexts`, optionally `ground_truth`.

**How RAGAS computes it.** For each retrieved passage in rank order, judge (via LLM prompt) whether it is relevant to the question. Compute the *mean average precision*-style aggregate that rewards putting relevant passages before irrelevant ones. RAGAS's exact formulation is `context_precision@k = sum over ranks of (precision_at_rank * relevance_indicator) / total_relevant`.

**What a good value looks like.** 1.0 means every retrieved passage is relevant *and* they are ranked ideally. 0.5 typically means either the retriever is bringing in noise, or the top-ranked passages are less relevant than the lower-ranked ones (a re-ranking signal).

**What it does not tell you.** Whether the retriever *missed* passages that were required to answer the question. That is context recall's job.

### 4. Context recall

**Question the metric answers.** "Did the retriever bring back the passages you would need in order to answer the question?"

**Inputs.** `question`, `contexts`, `ground_truth`. (Recall requires knowing the correct answer; this is the one core metric that is not reference-free.)

**How RAGAS computes it.** Break the ground-truth answer into atomic claims. For each claim, judge whether the retrieved context contains sufficient information to support that claim. Score = (fraction of ground-truth claims covered by the retrieved context).

**What a good value looks like.** 1.0 means every claim the ground-truth answer makes is supported by *something* in the retrieved context. Low values are the primary retrieval-failure signal — the corpus may contain the right passage, but the retriever did not surface it.

**What it does not tell you.** Whether the generator *used* the relevant passages (that's faithfulness) or whether the answer is coherent (that's answer relevancy).

## How the four metrics map to failure modes

The point of running the four together is that they *decompose* the pipeline. A given failure mode lights up a specific metric pattern:

| Symptom | Faithfulness | Answer relevancy | Context precision | Context recall |
| --- | --- | --- | --- | --- |
| Retriever misses the right passage | (may be high — grounded in wrong context) | (may be high) | (may be high — irrelevant-but-consistent context) | **low** |
| Retriever brings back irrelevant passages | (varies) | (varies) | **low** | (may be fine if the right passage also got in) |
| Generator hallucinates beyond context | **low** | (may be high) | (unaffected) | (unaffected) |
| Generator ignores context and uses priors | **low** | (may be high) | (unaffected) | (unaffected) |
| Answer rambles / off-topic | (varies) | **low** | (unaffected) | (unaffected) |

That table is the value proposition of RAGAS-style metrics. A single "correctness" number would give you "the pipeline is broken"; the four-metric decomposition tells you *where in the pipeline* to look.

Read the table again the other way: a *single* low metric is a strong signal, but *high on all four* is not a guarantee of correctness. Faithfulness + answer relevancy + context precision + context recall = 1.0 for four samples is compatible with the pipeline being "grounded in a retrieved but factually wrong corpus." The metrics measure pipeline consistency, not truth. Chapter 5 makes that argument in detail.

## TruLens's slightly different vocabulary

TruLens formalizes the same pattern under the label **RAG Triad**:

1. **Context relevance** — parallel to RAGAS's context precision. Is the retrieved context relevant to the question?
2. **Groundedness** — parallel to RAGAS's faithfulness. Is the answer grounded in the retrieved context?
3. **Answer relevance** — parallel to RAGAS's answer relevancy. Does the answer address the question?

TruLens exposes these as *feedback functions* attached to trace spans; they can be computed live in production over any RAG chain instrumented with the TruLens SDK. Context recall as a fourth metric is not part of the core triad because it requires ground truth; TruLens supports it via a separate `groundtruth_agreement` family.

The choice between RAGAS and TruLens is largely a workflow choice — RAGAS is a Python-package-based batch-eval tool tightly integrated with LangChain / LlamaIndex output shapes; TruLens is an observability tool that instruments a live application and computes feedback over recorded traces. Both are commonly used together: RAGAS for release-time offline evals over a curated gold set, TruLens for live-traffic monitoring.

## Judge choice: don't take the default

Every metric above is computed by an LLM judge. Every judge assumption from mod-105 applies — the judge is a measurement instrument with systematic bias, needs calibration to humans on the specific rubric, and drifts across releases if the judge is a hosted model.

Practical defaults:

- **Use a strong hosted judge** (frontier model) for the metric computation. RAGAS defaults to GPT-3.5 / GPT-4 in older versions; the current library accepts any LangChain LLM. TruLens accepts any provider.
- **Do not use the same model** as generator and judge if you care about the number. Faithfulness scored by the generator model is contaminated by self-preference (mod-105 Chapter 4).
- **Calibrate the judge on your task.** Faithfulness on financial documents behaves differently from faithfulness on customer-support tickets; a 100-item human-labelled slice with κ against the judge lets you *report* the judge's reliability alongside the metric value. Skipping this step turns the RAGAS number into "an LLM's opinion of an LLM's grounding," which does not gate deploys defensibly.
- **Version-pin the judge.** A model update to your judge changes the metric distribution silently; treat a judge model swap as a metric version bump.

## Reference-free vs. reference-full: pick a mix

Faithfulness, answer relevancy, and context precision are reference-free — they run against any RAG trace with no gold labels. That makes them attractive for continuous monitoring: run them over sampled production traffic and dashboard the distributions. Context recall (and any correctness-style metric like `answer_correctness` in RAGAS or `groundtruth_agreement` in TruLens) is reference-full — it needs a curated gold set.

The production pattern most teams settle on:

- **A curated gold set of ~200–500 items** with question + reference answer + reference context. Runs all four core metrics on every release; failing thresholds gate deploys.
- **Sampled production traffic** with the three reference-free metrics only, dashboarded per-day. Trend breaks are the alerting signal; per-item review of low-scoring cases is the debugging loop.

Both are cheap enough to run continuously if the judge is not a frontier model; use a mid-tier judge (or a dedicated one like Prometheus) for the traffic monitoring and reserve the frontier judge for the gold-set eval and for cases the traffic monitor flagged.

## The four objects, translated for RAG

- **Model adapter.** Wraps the *pipeline* — question in, `{answer, contexts}` out. The eval does not care which retriever + generator produced the pair as long as both are exposed.
- **Task definition.** A `question`, an optional `ground_truth` answer, and an optional `ground_truth_context`. Which metrics you can compute depends on which of these are present.
- **Request type.** Generation for the RAG pipeline; generation for the judge.
- **Scorer.** A per-metric judge chain plus a metric aggregator. The aggregator surface for RAGAS is a per-metric mean plus a `harmonic_mean` combined score; for TruLens it is the feedback function values attached to each trace span, aggregated by whichever dashboard query you run.

## Summary

RAG evaluation cannot be done as a single-turn correctness check; the pipeline has a hidden intermediate (retrieved context) that the metric has to peer into. RAGAS and TruLens both instrument the pipeline with a small family of judge-based metrics: *faithfulness* (is the answer grounded in the retrieved context), *answer relevancy* (does the answer address the question), *context precision* (are the retrieved passages relevant to the question), and *context recall* (are the passages required to answer the question actually in the retrieval, requires ground-truth). Together they decompose failure modes across the pipeline slots — a single low metric localizes the fault to retriever, ranker, or generator — but they measure pipeline *consistency*, not truth, and reading them as a correctness dashboard misuses them. The next chapter walks the specific failure modes of each metric and the discipline around them.
