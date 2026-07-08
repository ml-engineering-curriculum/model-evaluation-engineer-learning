# Prerequisites for the Model Evaluation Engineer track

This curriculum starts at **level 30 (deep specialist — peer to Senior ML Engineer)**. It does not re-teach the build-altitude Machine Learning Engineer foundation or the build-altitude LLM-eval practitioner foundation. Before working through it, you should already be comfortable with the items below. The [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) track (level 20) is the canonical place to acquire the ML foundation, [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (level 25) the canonical place to acquire build-altitude LLM/agent eval-harness work, and [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) (level 10) below that for engineering craft.

## Required

- **Intermediate Python** — functions, classes, modules, virtual environments, packaging (`uv` / `poetry`), structured logging, unit testing, and the ability to read a moderately complex open-source library (you will read `lm-evaluation-harness` internals).
- **Classical ML evaluation fluency** — confusion-matrix-derived metrics (precision / recall / F1, accuracy, MCC), ROC / PR / log loss / Brier, train / dev / test discipline, and the sklearn-style workflow at the level of `ml-engineer-learning` mod-104 / mod-105. mod-103 of this track *deepens* this, not introduces it.
- **PyTorch working familiarity** — load a model, run inference, read logits, compute log-probs. You will not implement training loops here, but mod-104 (lm-evaluation-harness) and mod-107 (multimodal) require comfort reading PyTorch.
- **Hugging Face `transformers` working familiarity** — load a model and tokenizer, run `generate()`, work with `AutoModelForCausalLM`, and understand chat templates well enough to debug a prompt-format sensitivity bug.
- **Basic statistics and probability** — distributions, expectations, conditional probability, the meaning of a confidence interval, the meaning of a p-value, and the bootstrap intuition. mod-101 *deepens* this — it does not introduce it.
- **At least one LLM API used in anger** — OpenAI / Anthropic / open-source LLM via an API. You will be evaluating these.
- **Experiment tracking** — at least one of MLflow / Weights & Biases used on a small project.
- **Linux command line and Git** — assumed at the level of `ai-infra-junior-engineer-learning`.
- **SQL and a Pandas / Polars equivalent** — eval data warehouses and slice reports live here.

## Recommended

- **One eval pipeline shipped at any depth** — even a Colab `lm-eval-harness` run on a single benchmark counts. You will move much faster if you have already felt the prompt-format sensitivity and dataset-format pain once.
- **One human-annotation project at any scale** — annotated 50 examples for a labelling pipeline, ran a Likert study, or written annotation instructions for somebody else. Module 106 will land with much more depth if you have.
- **Exposure to A/B testing** — a college course, a previous job, or the Kohavi-Tang-Xu chapters used as the canonical reference in mod-110.
- **One toy fairness audit** — Fairlearn or Aequitas on a public dataset. Module 103 covers methodology; pre-existing intuition for the impossibility results helps.
- **API and CLI credits** — a budget of \$50–\$100 in frontier-API credits for judge work in modules 105 / 107 / 108 and a handful of GPU hours (rented A10G / L4 class is fine) for harness work in module 104.
- **Hugging Face account** — datasets, models, and `lm-evaluation-harness` tasks integrate cleanly with an authenticated account.

## Out of scope as prerequisites

You do **not** need to know any of the following before starting; they are either taught in this curriculum or owned by a peer/upstream track:

- `lm-evaluation-harness`, HELM, OpenAI evals, Inspect, Promptfoo, DeepEval, RAGAS, TruLens, Arize Phoenix, Langfuse, W&B Weave — taught here.
- HarmBench / Cybench / METR tasks / WebArena / GAIA / AgentBench / SWE-bench — taught here.
- LLM-as-judge rubric design and bias controls (position / length / self-preference) — taught here.
- Bootstrap CIs, Wilson intervals, McNemar, FDR control — taught here.
- CUPED variance reduction and confidence sequences — taught here.
- NIST AI RMF / EU AI Act / ISO 25059 mapping for release gates — taught here.
- **Post-training depth** (SFT / PEFT / RLHF / DPO) — owned by [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning). This curriculum evaluates *outputs* of those pipelines.
- **Distributed-training platform engineering** — owned by [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning).
- **App-altitude LLM/agent eval-harness practitioner work** — owned by [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning).
- **Release-assurance / governance shape** — owned by [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) and [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning).
- **Alignment-risk methodology and red-team data generation** — owned by [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning).
- **Deep ML/AI security** — owned by [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning).
- **LLM application / RAG / NLP engineering depth** — owned by the respective peer specialist tracks linked from `CURRICULUM.md`.

## Hardware and credits

- Modules 101 and 103 can be done on a CPU laptop with a small classical-ML dataset.
- Module 104 (`lm-evaluation-harness`, HELM, Inspect) expects a 16-GB GPU or rented equivalent (A10G / L4 class). HELM-style scenario reproduction is the most expensive task; budget a few rented GPU-hours.
- Module 105 (LLM-as-judge) is API-bound — plan \$30–\$60 in frontier-API credits across two judge tiers plus an open-source judge.
- Module 107 (RAG / multimodal eval) reuses the module 104 GPU plus an open vision-language model (e.g. a 7B-class VLM) — a 24-GB GPU or rented equivalent for the multimodal exercise.
- Module 108 (agent eval) is partly sandbox-bound (Docker for SWE-bench-Lite, WebArena's hosted sandbox) and partly API-bound — plan \$30–\$60 in API credits and Docker on a machine that can spare 16 GB of RAM.
- Module 109 (safety eval) is API-bound and small-set human-eval-bound; \$20–\$40 of credits is enough for an awareness-level walkthrough. **Do not collect or distribute attack content.**
- Module 110 (production eval / regression) needs no GPU; a CSV of synthetic A/B traffic and a notebook are sufficient for the worked exercises.
- Module 111 (eval platform) needs a small cloud account for object storage and a managed Postgres or DuckDB — under \$10 for the lab.

<!-- needs-research: confirm the hardware-and-credits budget against the live posting evidence on the next autonomous research cycle; if postings consistently demand multi-GPU lm-eval-harness runs or large WebArena sweeps, raise the floor on the recommended-hardware list. -->
