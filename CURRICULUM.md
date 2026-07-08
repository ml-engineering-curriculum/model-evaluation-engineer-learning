# Model Evaluation Engineer Curriculum

**Role level:** 30 (deep specialist — peer to Senior ML Engineer on the ladder)
**Status:** planned — modules and projects below are the planned scope authored from [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Lessons and projects will be drafted by subsequent autonomous content cycles.

## Overview

This track teaches model evaluation engineering end-to-end for a deep specialist: validity and statistical methodology → benchmark engineering → classical-ML eval depth → LLM benchmark harnesses → LLM-as-judge platforms → human evaluation → generative and multimodal evaluation → agent and tool-use evaluation → safety and red-team measurement → production evaluation and regression detection → eval platform engineering → release-gate systems design and the governance interface.

It is a **specialist track at level 30** — it inherits ML / PyTorch / sklearn-style eval fundamentals from the build-altitude [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) track. Cross-modality coverage (classical, LLM, multimodal, agentic) and platform-altitude eval engineering are the differentiators against the peer `ai-eval-engineer-learning` (AI Engineering family, level 25 — app-side practitioner) and `ai-evaluation-engineer-learning` (Governance family, level 25 — release-assurance shape).

Total planned commitment: **175 hours** across 12 modules + **135 hours** across 3 projects = **~310 hours**.

## Ownership rule

Following the project-wide ownership rule, this curriculum:

- **Owns** evaluation engineering end-to-end at depth — validity & statistical methodology, benchmark engineering, harness engineering across modalities, human / LLM-judge calibration, safety & red-team measurement, production regression detection, eval platform internals, and release-gate systems design.
- **Defers down** to [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (level 20) for the build-altitude practitioner foundation (Python / PyTorch / sklearn / FastAPI / Docker / MLflow / experiment tracking / classical eval basics) and to [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) for engineering-craft prerequisites.
- **Defers sideways** to [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (level 25, AI Engineering family) for build-altitude LLM/agent eval-harness practitioner work inside product pipelines; to [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (level 25, Governance family) for release-assurance / governance shape (audit trails, regulator interface); to [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) and [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning) for post-training and training-platform depth; to [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning) / [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning) / [`nlp-engineer-learning`](https://github.com/ml-engineering-curriculum/nlp-engineer-learning) for their respective specialist domains; to [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning) for alignment-risk methodology depth.
- **Defers up** to [`staff-ml-engineer-learning`](https://github.com/ml-engineering-curriculum/staff-ml-engineer-learning) and [`principal-ml-engineer-learning`](https://github.com/ml-engineering-curriculum/principal-ml-engineer-learning) for architectural and leadership scope; to [`ai-infra-ml-platform-learning`](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning) for self-service ML platform engineering; to [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) for deep ML/AI security.
- **Links out** to [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning) and [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning) for governance / compliance / model-card review depth.

See [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) for the requirements-to-coverage map and the cited public references the catalog is grounded in.

## Module Plan

| Module | Title | Hours | Status |
|---|---|---|---|
| mod-101-evaluation-foundations | Evaluation Foundations: Validity, Estimation, and the Math of Measurement | 12 | planned |
| mod-102-benchmark-engineering | Benchmark Engineering: Constructing, Versioning, and Decontaminating Datasets | 16 | planned |
| mod-103-classical-ml-eval-depth | Classical ML Evaluation Depth: Calibration, Slicing, and Fairness | 14 | planned |
| mod-104-llm-benchmark-harnesses | LLM Benchmark Harnesses: lm-eval-harness, HELM, OpenAI Evals, Inspect | 16 | planned |
| mod-105-llm-as-judge-platforms | LLM-as-Judge Platforms: Rubrics, Bias Controls, and Calibration to Humans | 15 | planned |
| mod-106-human-evaluation | Human Evaluation: Annotator Workflows, Agreement, and Gold Sets | 12 | planned |
| mod-107-generative-and-multimodal-eval | Generative and Multimodal Evaluation: Code, Math, RAG, Vision | 16 | planned |
| mod-108-agent-and-tool-eval | Agent and Tool-Use Evaluation: Trajectories, Sandboxes, and Partial Credit | 15 | planned |
| mod-109-safety-and-red-team-eval | Safety and Red-Team Evaluation | 15 | planned |
| mod-110-production-eval-regression | Production Evaluation and Regression Detection | 16 | planned |
| mod-111-eval-platform-engineering | Eval Platform Engineering: Eval-as-a-Service | 16 | planned |
| mod-112-eval-systems-design | Eval Systems Design and the Governance Interface | 12 | planned |

## Project Plan

| Project | Title | Hours | Status |
|---|---|---|---|
| project-101-benchmark-engineering-capstone | Benchmark Engineering Capstone: A Decontaminated, Versioned, Per-Slice Benchmark for a Real Domain | 35 | planned |
| project-102-llm-judge-calibration-study | LLM-as-Judge Calibration Study: Validate a Judge Against Human Gold for a Production Task | 40 | planned |
| project-103-eval-platform-capstone | Eval Platform Capstone: Eval-as-a-Service with Release Gates for a Stated Product Pipeline | 60 | planned |

## Module summaries

### mod-101 — Evaluation Foundations
Validity (construct / internal / external), point estimates with non-asymptotic confidence intervals (Wilson, paired bootstrap), paired-comparison tests (McNemar, sign test, paired bootstrap), multiple-comparison correction (Benjamini-Hochberg FDR) when reporting many slices or many metrics, and how to read an eval report for validity, sampling, statistical-power, and contamination weaknesses.

### mod-102 — Benchmark Engineering
Source raw data with documented licensing and provenance; gold-standard labelling with IAA and adjudication; contamination detection (n-gram overlap, embedding overlap, log-likelihood probes); benchmark versioning (dataset / task / scorer hashes); public / private holdout management; canary sets; deprecation policy.

### mod-103 — Classical ML Eval Depth
Per-slice / per-segment metrics reporting; calibration measurement (ECE, reliability diagrams) and recalibration (Platt, isotonic, temperature); operating-point selection under cost asymmetry; fairness measurement (demographic parity, equalized odds, equal opportunity) with Fairlearn / Aequitas including the impossibility results; regression and ranking eval (RMSE, MAE, MAPE, R², NDCG, MAP) with subgroup breakdowns.

### mod-104 — LLM Benchmark Harnesses
Register and run new tasks in EleutherAI `lm-evaluation-harness` (log-likelihood and generation); reproduce HELM scenario results; stand up Inspect (UK AISI) evals with solver/scorer plumbing; build OpenAI evals-style registry entries with model-graded scoring; diagnose prompt-format sensitivity, log-prob-vs-generation gap, and per-task reproducibility (seeds, decoding params, dataset hash).

### mod-105 — LLM-as-Judge Platforms
Absolute and pairwise rubric design with explicit anchors; position-bias and length-bias controls (swap-and-average, length normalisation); self-preference and verbosity-bias diagnostics; calibration against a human gold (Cohen's kappa, Spearman / Kendall); Arena-style ELO / Bradley-Terry rating; dedicated judge models (Prometheus 2, JudgeLM); judge-tier routing across cost and quality.

### mod-106 — Human Evaluation
Annotator recruitment and training; instruction design and pilot annotation; inter-annotator agreement (Cohen's / Fleiss' kappa, Krippendorff's alpha); adjudication; gold-set rotation; side-by-side UX with attention-check items; build-vs-buy across Scale / Surge / Mercor / Prolific / Argilla / Snorkel / in-house with data-residency trade-offs explicit.

### mod-107 — Generative and Multimodal Eval
Pass@k functional correctness with sandboxed execution and timeout policy; math reasoning with answer-extraction robustness and CoT rubrics; RAG eval (RAGAS / TruLens — faithfulness, answer relevancy, context precision/recall) with failure-mode reasoning; multimodal eval (image-text alignment, VQA, vision-grounded instruction following); a product-shaped eval suite that reflects a real surface rather than a leaderboard collage.

### mod-108 — Agent and Tool-Use Eval
Trajectory-level scoring (per-step tool correctness, final-answer correctness, cost / latency / step-count budgets); SWE-bench / SWE-bench Verified reproduction; WebArena / GAIA / AgentBench runs with sandbox isolation and deterministic replay; partial-credit rubric design and human-gold audit; Inspect's agent harness end-to-end.

### mod-109 — Safety and Red-Team Eval
Refusal / over-refusal / jailbreak-resistance measurement against HarmBench-style standardized suites; dangerous-capability evals (cyber / autonomy / bio) at the awareness level, mapped to Anthropic RSP, OpenAI Preparedness, and UK AISI methodology; bias / toxicity / fairness measurement with sensitivity to rater bias; prompt-injection robustness; safety-eval section of a model card written so it satisfies governance / risk partners without leaking attack payloads.

### mod-110 — Production Eval and Regression Detection
Offline regression suites with pass/fail gates mapped to product SLOs and safety policy; shadow / dark-launch comparisons; A/B with CUPED variance reduction and pre-registered hypotheses; sequential testing / confidence sequences for safe continuous monitoring; production observability integration (Arize Phoenix, Langfuse, W&B Weave) with drift and judge-drift alerts; MLPerf-style inference benchmarking at the serving altitude (TTFT, TPOT, throughput vs. accuracy floor).

### mod-111 — Eval Platform Engineering
Eval-as-a-service architecture with versioned task / dataset / judge / prompt registry; multi-runner orchestration (lm-eval-harness + Inspect + OpenAI evals + custom runners) behind a single plane; parallelism and cost controls at platform scope; eval data warehouse with traceable lineage (model / dataset / judge / prompt / decoding / seed hashes); CI integration and SLA / SLO for the eval system itself.

### mod-112 — Eval Systems Design and Governance Interface
Product spec → release-gate eval plan with explicit pass thresholds and rollback criteria; model card composed from eval evidence; mapping to NIST AI RMF (Measure / Manage), ISO/IEC 25059 quality dimensions, and EU AI Act high-risk obligations where applicable; build-vs-buy across eval platforms (in-house, Arize, Langfuse, Weave, OpenAI evals, vendor-hosted); eval budget defence across cost / wall-clock / coverage.

## Assessment

Each module ships **1 quiz** plus the exercises and a lab listed in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Each project ships a portfolio-grade README, a reproducibility bundle (config + dataset hashes + judge hashes + seeds), and an explicit rubric covering validity, benchmark engineering, statistical methodology, safety coverage, platform integration, and documentation.

## Where to go after this curriculum

- **`senior-ml-engineer-learning`** — peer at the same level (generalist counterpart on the ladder).
- **`staff-ml-engineer-learning` / `principal-ml-engineer-learning`** — next steps on the ML engineering ladder for architectural / leadership scope.
- **`fine-tuning-engineer-learning`** — peer specialist that consumes this curriculum's eval methodology as release gates.
- **`ai-eval-engineer-learning`** — peer for app-altitude LLM/agent eval-harness practitioner work.
- **`ai-evaluation-engineer-learning`** — peer for release-assurance / governance shape.
- **`ai-risk-engineer-learning`** — peer specialist for alignment-risk methodology and red-team data generation.
- **`ai-infra-ml-platform-learning`** — for self-service ML platform engineering hosting eval-as-a-service.
- **`ai-infra-security-learning`** — for deep ML/AI security, including eval-set integrity and judge supply-chain.
- **`ai-governance-analyst-learning`** / **`head-of-ai-governance-learning`** — for governance / compliance / regulator-interface depth.

<!-- needs-research: backfill industry-frequency evidence into JOB_REQUIREMENTS.md once the autonomous research loop runs with WebSearch / WebFetch exercised against live job boards; demote any module or exercise whose underlying requirement does not show up in ≥3 in-window postings. -->
