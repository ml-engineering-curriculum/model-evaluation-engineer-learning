# mod-107-generative-and-multimodal-eval: Generative and Multimodal Evaluation: Code, Math, RAG, Vision

**Estimated effort:** 16 hours

The previous three modules taught you how to *run* an eval (mod-104 harnesses), how to *use a model as a scorer* (mod-105 LLM-as-judge), and how to *use humans as scorers* (mod-106 human evaluation). This module is about the four families of task where those scorers stop being interchangeable: code generation, math reasoning, retrieval-augmented generation, and multimodal (vision-language). Each family has a canonical measurement instrument the field has agreed on — pass@k with sandboxed execution, extraction plus optional chain-of-thought rubrics, RAGAS/TruLens metric families, VQA-style accuracy with explicit modality coverage — and each instrument has documented failure modes that turn the reported number into noise if you run the instrument without them.

The chapters walk each instrument in turn, spending a full chapter on the RAG failure-modes catalogue (which is where most homegrown RAG dashboards silently mislead their readers) and a full chapter on the multimodal coverage argument (which is the eval-design question specific to vision). The last chapter is the composition step: given the four instruments in this module plus everything from mod-104 through mod-106, how do you build an eval suite for *your* product surface without falling into the leaderboard-collage antipattern.

## Learning objectives

- Evaluate code-generation models with pass@k functional correctness, including sandboxed execution and timeout policy.
- Evaluate math reasoning (GSM8K, MATH) with answer-extraction robustness and chain-of-thought rubrics.
- Evaluate retrieval-augmented generation with RAGAS / TruLens (faithfulness, answer relevancy, context precision/recall) and reason about each metric's failure modes.
- Evaluate multimodal models (image-text alignment, VQA, vision-grounded instruction following) and reason about modality coverage.
- Compose a cross-capability eval suite that reflects a stated product surface, not a leaderboard collage.

## Lecture chapters

1. [`01-why-generation-and-multimodal-break-string-match.md`](01-why-generation-and-multimodal-break-string-match.md) — the four families where reference-string equality is the wrong shape, the four instruments the field agreed on, and the four-objects taxonomy translated into each family.
2. [`02-pass-at-k-and-the-sandboxed-execution-loop.md`](02-pass-at-k-and-the-sandboxed-execution-loop.md) — pass@k with the unbiased Chen et al. 2021 estimator, sandbox properties (process, filesystem, network, and resource isolation), timeout policy as semantics not plumbing, sampling temperature by `k`, and HumanEval / MBPP / HumanEval+ / LiveCodeBench versioning.
3. [`03-math-reasoning-extraction-and-chain-of-thought-rubrics.md`](03-math-reasoning-extraction-and-chain-of-thought-rubrics.md) — GSM8K and MATH, the layered extraction pipeline (`\boxed{}`, `#### N`, prose fallback, `sympy` CAS equivalence), CoT rubrics that score process not just outcome, format-compliance-rate as a metric, and self-consistency `acc@maj-N`.
4. [`04-rag-metrics-with-ragas-and-trulens.md`](04-rag-metrics-with-ragas-and-trulens.md) — the RAG pipeline as an instrumentation surface, the four RAGAS metrics (faithfulness, answer relevancy, context precision, context recall) with what each judge chain actually computes, and TruLens's RAG Triad framing.
5. [`05-rag-failure-modes-where-each-metric-misleads.md`](05-rag-failure-modes-where-each-metric-misleads.md) — where each RAG metric misleads: faithfulness's omission blind spot and "grounded in the wrong context" trap, answer-relevancy's topicality-vs-correctness confusion, context precision's sensitivity to `k`, context recall's dependence on ground truth *contexts* not just *answers*, and the debugging protocol that walks the metrics in order.
6. [`06-multimodal-eval-vqa-alignment-and-modality-coverage.md`](06-multimodal-eval-vqa-alignment-and-modality-coverage.md) — the three multimodal instruments (VQA accuracy, image-text alignment, grounded instruction following) with their canonical benchmarks (VQAv2, MMMU, ChartQA, DocVQA, POPE, RefCOCO), and the modality-coverage argument that separates multimodal eval design from text-only eval design.
7. [`07-composing-a-product-shaped-eval-suite.md`](07-composing-a-product-shaped-eval-suite.md) — the leaderboard-collage antipattern, the surface-to-instrument construction pattern, weighting components by traffic or impact, sentinel items, and the writeup discipline (product surfaces, capability map, weighting rationale, per-instrument configuration, coverage gaps, reproducibility manifest).

## Exercises

Five hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after the chapters it depends on.

- [`exercise-01-pass-at-k-with-sandbox-execution.md`](exercises/exercise-01-pass-at-k-with-sandbox-execution.md) — build a functional-correctness eval end-to-end: sandboxed executor with timeout and resource limits, the Chen-et-al. unbiased pass@k estimator, and a reproducible HumanEval / HumanEval+ run with the scorer probed against seeded good, bad, infinite-loop, and OOM samples.
- [`exercise-02-math-eval-with-robust-extraction.md`](exercises/exercise-02-math-eval-with-robust-extraction.md) — a layered extractor pipeline for GSM8K and MATH, with a `sympy` CAS-equivalence layer for MATH, per-layer contribution reporting, and a sampled CoT rubric on 200 items graded per-criterion.
- [`exercise-03-rag-eval-with-ragas-and-trulens.md`](exercises/exercise-03-rag-eval-with-ragas-and-trulens.md) — stand up a minimal RAG pipeline against a small corpus, run all four RAGAS metrics plus the TruLens RAG Triad, calibrate against a 50-item human-labelled slice with κ, and produce a failure-modes writeup that identifies at least three items where a specific metric misled.
- [`exercise-04-multimodal-vqa-and-grounding-eval.md`](exercises/exercise-04-multimodal-vqa-and-grounding-eval.md) — evaluate a vision-language model on a VQA benchmark, add a POPE-style hallucination probe on 100 items, add a bespoke slice from a modality the benchmark does not cover, and produce a coverage-first writeup.
- [`exercise-05-product-shaped-eval-suite-design.md`](exercises/exercise-05-product-shaped-eval-suite-design.md) — take a stated product surface (from a menu or your own), build the surface-to-instrument mapping, weight the components, curate sentinel items, and produce the suite writeup with an explicit coverage-gaps section.

Reference solutions live in the paired [`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for primary references — the Codex / HumanEval paper (Chen et al. 2021), MBPP (Austin et al. 2021), HumanEval+ / EvalPlus (Liu et al. 2023), LiveCodeBench (Jain et al. 2024), GSM8K (Cobbe et al. 2021), MATH (Hendrycks et al. 2021), *Let's Verify Step by Step* (Lightman et al. 2023), self-consistency (Wang et al. 2022), RAGAS (Es et al. 2023) and TruLens documentation, VQAv2 (Goyal et al. 2017), MMMU (Yue et al. 2024), ChartQA / DocVQA / POPE, CLIPScore (Hessel et al. 2021), and the harness / library repositories behind all of the above.
