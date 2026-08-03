# RAG Failure Modes: Where Each Metric Misleads

The previous chapter walked what RAGAS and TruLens metrics *measure*. This chapter walks what they miss. Every metric in that family is a proxy — a computable question the judge can answer, standing in for a harder question no single number captures. The proxies are useful precisely because they are cheap and decompose the pipeline into named slots. They mislead precisely because their failure modes are systematic, and a team that publishes a RAG dashboard without a failure-modes section is telling a plausible story that will not survive contact with a real regression.

The failure modes are worth walking one metric at a time. Each has (a) the specific pipeline state where the metric lies, (b) the wrong conclusion someone reading the dashboard will draw, and (c) the mitigation.

## Faithfulness: the omission blind spot

Faithfulness measures the fraction of claims *in the answer* that are supported by the retrieved context. It does not measure whether the answer *includes* the claims that the context supports. That asymmetry is the largest single trap in the metric.

**The failing case.** A user asks "What are the side effects of drug X?" The retrieved context lists five side effects: dizziness, nausea, fatigue, insomnia, and (buried in a paragraph about interactions) a rare but serious risk of cardiac arrhythmia. The generator writes: "The side effects include dizziness, nausea, fatigue, and insomnia." Faithfulness = 1.0 (every claim in the answer is grounded), and the answer omitted the medically significant side effect.

**The wrong conclusion.** "Faithfulness is high, so the answer is trustworthy."

**Why it happens.** The metric is claim-in-answer-supported-by-context, not claim-in-context-covered-by-answer. RAG generators tuned for concision or safety often drop hedged, technical, or negative material. The metric applauds them for it.

**Mitigations.**

- Report *coverage* alongside faithfulness. Coverage is the reverse — for each atomic claim in the *reference answer*, is it present in the model's answer? This is what RAGAS calls `answer_correctness` in some formulations and Anthropic's context-relevance-and-coverage internal metrics track separately. Coverage requires a reference; on unlabelled traffic, a proxy is *context coverage*: for each atomic claim in the retrieved context that the judge deems "salient to the question," is it in the answer? Both are noisy but they surface omissions faithfulness cannot.
- Look at low-token answers manually. Faithfulness = 1.0 on a suspiciously short answer is a specific red flag.
- Design the eval set to *include* omission-trap items: questions whose ground-truth answer requires a specific caveat or negative statement that a cautious generator might strip.

## Faithfulness: the "grounded in the wrong context" trap

**The failing case.** The retriever pulls a passage from a related but incorrect source — say a passage about a different drug with the same trade name, or a wiki article about a homonym company. The generator faithfully summarizes that passage. Faithfulness = 1.0, and the answer is wrong.

**The wrong conclusion.** "The answer is grounded, so the retriever must have found the right thing."

**Why it happens.** Faithfulness scores answer-vs-retrieved-context. It knows nothing about whether the retrieved context was itself correct.

**Mitigations.**

- Pair faithfulness with *context recall* on labelled items — low context recall + high faithfulness = grounded in the wrong context.
- On unlabelled traffic, use a *context-vs-corpus* consistency check: sample a claim from the retrieved passage and query the corpus for contradicting passages. This is heavy but is the pattern used in production retrieval-QA monitoring.
- Design the eval set to include *ambiguity items* (drug name shared by two drugs, company name shared by two companies) and monitor for the pipeline picking the wrong branch.

## Answer relevancy: rewards topicality, not correctness

**The failing case.** "What is the capital of Australia?" → "Sydney is the capital of Australia." Answer relevancy = 0.95 (the answer would be a good response to a question very close to the original). Correctness = 0.

**The wrong conclusion.** "The answer is on-topic and complete."

**Why it happens.** Answer relevancy is a topicality metric — it measures whether the answer *sounds like* it's addressing the question. A confident wrong answer with on-topic vocabulary scores as well as a correct answer.

**Mitigations.**

- Never read answer relevancy as a correctness metric. It is an *anti-rambling* metric — it catches "I don't have information about X, but let me tell you about Y" and "Great question! Here is some tangentially related content." Nothing more.
- Always pair with faithfulness or an outcome-based correctness measure. High relevancy + low faithfulness = fluent-but-hallucinated. High relevancy + high faithfulness + unknown correctness = still needs a ground-truth check.
- Especially watch relevancy on *refusal* items ("I cannot answer that"). A good refusal often scores *low* on relevancy because its answer is not shaped like an answer to the question; that is not a defect. Filter or scoring-adjust for refusals separately.

## Context precision: sensitive to top-k

**The failing case.** The retriever is set to `k = 10`, and the eval reports context precision = 0.4. The team sets `k = 3` and re-runs — precision jumps to 0.9. The team concludes "the retriever got better." It didn't; the tail three of the top-10 are usually irrelevant, but they were also usually harmless.

**The wrong conclusion.** "Higher precision = better retriever." The retriever hasn't changed; the *reporting cutoff* changed.

**Why it happens.** Precision is defined over the returned set. Shrinking the set inflates precision as long as the top ranks are stronger than the bottom ranks.

**Mitigations.**

- Report `context_precision@k` with `k` fixed to the value the *generator sees*. If the generator receives top-5, the eval metric uses top-5, not "the retriever's top-20."
- If you tune `k`, expect precision to move; the meaningful metric is a *joint plot* of context precision × answer quality × latency, not any single-`k` precision number.
- The signal you actually want is *whether re-ranking helps* — plot precision@1 vs precision@k. If precision@1 is high and precision@k is low, the retriever is finding the right thing and burying it; a re-ranker will help. If both are low, the retriever is missing the right thing and a re-ranker won't help.

## Context recall: needs ground truth *contexts*, not just ground truth *answers*

**The failing case.** The eval set has `question` and `ground_truth_answer` but no annotated `ground_truth_context`. RAGAS's context recall works by decomposing the ground-truth answer into claims and checking whether each claim is in the retrieved context. That is *not the same measurement* as "did the retriever return the passages a human labelled as relevant."

**The wrong conclusion.** "Context recall = 0.9 means the retriever is finding 90% of the relevant passages."

**Why it happens.** RAGAS-style context recall answers the question "does the retrieved context cover the claims the ground-truth answer makes?" It does not answer "which passages *should* the retriever have returned?" The two agree in the common case, but they diverge when:

- The ground-truth answer synthesizes across passages, and any of the passages would support the same claim. The metric accepts partial coverage that a strict passage-recall measure would flag.
- The ground-truth answer omits a claim that would have been supported by an important passage; the retriever legitimately returns that passage but the metric can't see it as coverage.

**Mitigations.**

- Where you can afford it, annotate `ground_truth_context` at the passage level — a set of `passage_id`s that a human considers relevant to the question. Compute the classic IR metrics (recall@k, MRR, nDCG) against that annotation. This is more work than RAGAS-style context recall but is what an IR team would recognize as retrieval evaluation.
- Reserve RAGAS's context recall for cases where the ground truth is *only* an answer text — it is a defensible proxy in that setting.
- Pair the two when possible. A gap between passage-level recall and RAGAS-style context recall exposes cases where the retriever pulled a "correct enough" passage that the RAGAS judge accepted as coverage but that a human annotator would have marked as a miss.

## Judge-based metrics inherit judge bias — all of it

Every metric in this family is computed by an LLM judge. Every mod-105 failure mode applies:

- **Self-preference.** If your generator and your judge are the same model family, faithfulness and answer relevancy are inflated.
- **Length bias.** Longer answers get higher relevancy scores; longer retrieved contexts get lower precision scores. Neither is fully fixable without length-normalisation aware evaluation.
- **Position bias.** Not directly applicable (the metrics are absolute, not pairwise) but similar in shape: the judge weights earlier atomic claims more than later ones, so long answers that put important claims later get slightly under-scored.
- **Rubric drift.** Judge model updates change the metric distribution silently. A jump in dashboard faithfulness on the same eval set is either a model change (real signal) or a judge change (fake signal) — you cannot tell without pinning judge versions.

The mitigation is the mod-105 discipline: a small (50–100 item) human-labelled slice with periodic κ recomputation between judge and humans. Publish κ alongside every RAG metric. Do not let a dashboard show "faithfulness = 0.87" without "judge-human κ on this task = 0.71" next to it.

## Composite scores hide their inputs

Both RAGAS and several downstream dashboards expose a "RAG score" that combines the individual metrics — often harmonic mean or a weighted sum. The composite is useful for a single alert threshold. It is misleading everywhere else, because it collapses the pipeline decomposition that made the metrics useful in the first place. A composite of 0.85 could be `[faithfulness=1.0, relevancy=0.9, cp=0.9, cr=0.6]` (retrieval miss) or `[0.7, 0.9, 1.0, 0.8]` (generator hallucination). The action item is entirely different.

**Rule of thumb.** Alert on the composite; *debug on the components*. Never publish only the composite; publish the four (or three, for reference-free traffic monitoring) alongside it.

## What RAG metrics do *not* measure at all

Several things that matter for a production RAG system are simply outside the RAGAS/TruLens metric family:

- **Latency and cost.** The RAG pipeline can be slow and expensive; the eval metrics ignore both. A separate metric axis (P50 / P95 latency, tokens-per-answer) belongs on the same dashboard.
- **Refusal appropriateness.** RAG systems should refuse when the retrieved context doesn't support an answer. RAGAS metrics under-reward correct refusals (relevancy dips, faithfulness is undefined). A separate refusal-appropriateness axis — did the system refuse when it should have, and did it answer when it shouldn't have refused — belongs alongside.
- **Safety of retrieved content.** If the retriever pulls a passage containing PII, malware, or policy-violating content, no core RAGAS metric flags it. Safety eval (mod-109) is separate.
- **Corpus freshness.** A pipeline can retrieve a stale but faithfully-summarized passage, produce a wrong current-day answer, and score perfectly on RAGAS. Time-sensitive eval requires timestamped ground truth and a freshness metric.
- **User-perceived helpfulness.** Whether the answer *helps the user solve their problem* is only weakly correlated with any of the four metrics. Human eval (mod-106) remains the ceiling.

## The debugging protocol

When the eval dashboard shows a problem, the debugging protocol pushes through the metrics in order:

1. **Context recall low?** Retriever miss. Debug: is the corpus complete? Is the embedding model appropriate to the domain? Is chunk size too large? Fix retriever first.
2. **Context recall fine, context precision low?** Retriever brings noise. Debug: add a re-ranker, filter by score threshold, tune `k` down.
3. **Context precision fine, faithfulness low?** Generator ignores context. Debug: prompt-engineer the generator toward "cite only from context," check for prior-heavy models that need lower temperature or stronger grounding instructions.
4. **Faithfulness fine, answer relevancy low?** Generator rambles or hedges. Debug: prompt-engineer for concision, check refusal rate, look for a system-prompt drift.
5. **All four fine, correctness bad?** You are in one of the traps above — omission, grounded-in-wrong-context, or judge bias. Rerun on a human-labelled slice.

The protocol is the value of the decomposition. A single "correctness" number would stop at step 5.

## Summary

Each of the RAGAS/TruLens metrics is a defensible proxy with a specific and well-understood failure mode: faithfulness misses omissions and rewards "grounded in the wrong context," answer relevancy rewards topicality but not correctness, context precision moves with reporting `k` regardless of retriever quality, and context recall depends on whether ground-truth is an answer or a set of passages. All four inherit the biases of the underlying judge and should be published with a companion judge-vs-human agreement number. Composite RAG scores are useful for alerting and misleading for debugging; publish the components. Refusal appropriateness, latency, cost, safety, and freshness are *not* measured by this family at all and need separate axes on the same dashboard. The next chapter turns to multimodal, where the failure modes shift from proxy-inversion to *coverage* — the eval measures what the benchmark tests, and the benchmark tests a small slice of the visual world.
