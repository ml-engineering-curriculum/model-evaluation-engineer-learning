# Job Requirements — Model Evaluation Engineer

**Role level:** 30 (deep specialist — peer to Senior ML Engineer on the ladder)
**Track:** `model-evaluation-engineer-learning`
**Research window:** 2026-03-27 → 2026-06-25 (last 90 days)
**Today:** 2026-06-25

This file documents the requirements catalog used to seed the Model Evaluation Engineer curriculum. Raw normalized data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json).

## Status — bootstrap session, postings deferred

<!-- needs-research: collect ≥25 distinct in-window postings titled "Model Evaluation Engineer" / "LLM Evaluation Engineer" / "Foundation Model Evaluation Engineer" / "AI Model Evaluation Engineer" / "Benchmark Engineer" / "Model Quality Engineer" / "Evaluation Research Engineer" (NOT Senior / Staff / Principal modifiers — those will inherit this packet — and NOT generic ML / Data-Scientist / Research-Scientist / AI-Application / Trust-&-Safety titles, which are owned by peer tracks). Re-validate every requirement against the live evidence and demote any whose evidence stays empty. -->

This packet was authored in a bootstrap session **without an exercised WebSearch / WebFetch pass against live job boards** (WebSearch permission was not granted in-session). Per the project rules (*"Do not invent facts, incidents, or salary figures. Cite sources."*), the `postings` array in `.aicg/job-requirements.json` is intentionally empty — the curriculum will not claim to have analysed 25 live postings when none were fetched.

The autonomous research loop is expected to fill that gap on its next cycle. To keep the loop deterministic, this document grounds the requirements catalog in **authoritative public references** that publish what the role is hired against: the canonical open-source eval frameworks (EleutherAI lm-evaluation-harness, Stanford CRFM HELM, OpenAI Evals, UK AISI Inspect, Hugging Face Evaluate, RAGAS, Promptfoo, DeepEval, TruLens), the canonical eval research literature (HELM, MMLU, BIG-bench, HumanEval, IFEval, TruthfulQA, MT-Bench / Chatbot Arena, SWE-bench, WebArena, GAIA, AgentBench, HarmBench), the canonical statistical methodology references (Efron's bootstrap, Benjamini-Hochberg FDR, Kohavi A/B testing, CUPED, Howard confidence sequences, Guo on calibration), public regulatory and standards documents (NIST AI RMF, EU AI Act, ISO/IEC 22989 / 25059), public risk frameworks (Anthropic RSP, OpenAI Preparedness, UK AISI), and worked-example public model / system cards (Llama 3, Gemini, Claude, GPT system cards). Every requirement below cites at least one such reference, and every requirement is shaped so that posting-frequency evidence can be added underneath it without restructure.

## Methodology

1. Sourced the canonical task domains for a deep-specialist model evaluation engineer from public references — see `authoritative_references` in `.aicg/job-requirements.json`:
   - **Harnesses / frameworks**: lm-evaluation-harness (EleutherAI), HELM (Stanford CRFM), OpenAI Evals, Inspect (UK AI Safety Institute), Hugging Face Evaluate, Promptfoo, DeepEval, TruLens, Langfuse, Arize Phoenix, W&B Weave
   - **Benchmarks**: MMLU, BIG-bench, HumanEval, IFEval, TruthfulQA, GSM8K / MATH, MT-Bench, Chatbot Arena, SWE-bench, WebArena, GAIA, AgentBench, HarmBench, Cybench, METR autonomy tasks, MLPerf Inference (datacenter + LLM track)
   - **Statistical methodology**: Efron 1979 (bootstrap), Benjamini-Hochberg 1995 (FDR), Guo 2017 (calibration / ECE), Platt 1999 (probabilistic calibration), Kohavi-Tang-Xu (A/B testing), Deng 2013 (CUPED), Howard et al. 2018 (confidence sequences)
   - **LLM-as-judge**: Zheng 2023 (MT-Bench), Wang 2023 (LLM judges are not fair evaluators — bias literature), Zhu 2023 (JudgeLM), Kim 2024 (Prometheus 2)
   - **Safety / risk**: HarmBench (Mazeika 2024), GCG (Zou 2023), Cybench (Zhang 2024), MITRE ATLAS, Anthropic Responsible Scaling Policy, OpenAI Preparedness Framework, UK AISI public methodology
   - **Standards / regulation**: NIST AI RMF (Measure / Manage functions), EU AI Act (Regulation 2024/1689), ISO/IEC 22989, ISO/IEC 25059, Mitchell 2019 (Model Cards)
   - **Worked-example disclosures**: Llama 3 technical report (Meta), Gemini technical report (Google DeepMind), public Claude model cards and GPT system cards
2. Mapped each task domain to (a) the role on our level ladder that should own it primarily and (b) the curriculum module that covers it.
3. Applied the **ownership rule**: assign coverage to the lowest-level role that genuinely requires the skill, with higher levels linking back rather than duplicating fundamentals. Defer down to `ml-engineer` (level 20) for classical-eval and engineering fundamentals; defer sideways to `ai-eval-engineer` (AI Engineering family, level 25) for build-altitude LLM/agent eval harness practitioner work in product pipelines; defer to `ai-evaluation-engineer` (Governance family, level 25) for release-assurance / governance shape; defer to `ai-risk-engineer` for alignment-risk methodology depth and to `ai-governance-analyst` / `ai-infra-security` for governance and security depth respectively.
4. Flagged everything that has not yet been validated against in-window postings so the next cycle can demote any requirement whose evidence stays empty.

## Requirement themes → curriculum ownership

The table below lists each requirement theme, its planned owner per the level hierarchy, and the curriculum coverage path. **Freq** is intentionally blank for this cycle — it will be backfilled by the next research pass.

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | Evaluation methodology: validity, point estimates with CIs, paired tests, FDR control | <!-- needs-research --> | `model-evaluation-engineer` (this) | [`mod-101-evaluation-foundations`](lessons/mod-101-evaluation-foundations) |
| 2 | Benchmark engineering: sourcing, labelling with IAA, decontamination, versioning, canary sets | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-102-benchmark-engineering`](lessons/mod-102-benchmark-engineering) |
| 3 | Classical ML eval depth: calibration, slicing, fairness, threshold selection | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-103-classical-ml-eval-depth`](lessons/mod-103-classical-ml-eval-depth) |
| 4 | LLM benchmark harnesses: lm-eval-harness, HELM, OpenAI evals, Inspect, prompt-format sensitivity | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-104-llm-benchmark-harnesses`](lessons/mod-104-llm-benchmark-harnesses) |
| 5 | LLM-as-judge: rubric design, position / length / self-preference bias, calibration to humans, Arena-style ELO | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-105-llm-as-judge-platforms`](lessons/mod-105-llm-as-judge-platforms) |
| 6 | Human evaluation: annotator workflows, IAA, gold-set rotation, vendor choice | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-106-human-evaluation`](lessons/mod-106-human-evaluation) |
| 7 | Generative & multimodal eval: pass@k code, math, RAG (RAGAS / TruLens), multimodal | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-107-generative-and-multimodal-eval`](lessons/mod-107-generative-and-multimodal-eval) |
| 8 | Agent & tool eval: trajectory scoring, SWE-bench, WebArena / GAIA / AgentBench, METR tasks, Inspect agent harness | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-108-agent-and-tool-eval`](lessons/mod-108-agent-and-tool-eval) |
| 9 | Safety / red-team eval: refusal, jailbreak resistance (HarmBench), dangerous-capability evals, prompt-injection robustness, bias / toxicity | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-109-safety-and-red-team-eval`](lessons/mod-109-safety-and-red-team-eval) |
| 10 | Production eval & regression: offline gates, shadow, A/B with CUPED, sequential testing, drift / judge drift, MLPerf | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-110-production-eval-regression`](lessons/mod-110-production-eval-regression) |
| 11 | Eval platform engineering: registry, multi-runner orchestration, eval data warehouse, CI integration, SLO | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-111-eval-platform-engineering`](lessons/mod-111-eval-platform-engineering) + [`project-103-eval-platform-capstone`](projects/project-103-eval-platform-capstone) |
| 12 | Eval systems design & governance interface: release gates, model cards, NIST AI RMF / ISO 25059 / EU AI Act mapping, build-vs-buy | <!-- needs-research --> | `model-evaluation-engineer` | [`mod-112-eval-systems-design`](lessons/mod-112-eval-systems-design) |
| 13 | PyTorch / sklearn-eval / FastAPI / Docker / experiment tracking fundamentals | n/a — prerequisite | `ml-engineer` (level 20) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught |
| 14 | Build-altitude LLM/agent eval-harness practitioner work in product pipelines | n/a — peer track | `ai-eval-engineer` (level 25, AI Engineering family) | Linked out; this curriculum stays at methodology / platform altitude |
| 15 | Release-assurance / governance-shaped evaluation (audit trails, regulator interface) | n/a — peer track | `ai-evaluation-engineer` (level 25, Governance family) | This curriculum produces the evidence; governance shape is owned upstream |
| 16 | Post-training (SFT / PEFT / RLHF / DPO) depth | n/a — peer track | `fine-tuning-engineer` (level 30) | Out of scope — we evaluate the outputs |
| 17 | Distributed-training PLATFORM engineering (multi-tenant schedulers, NCCL/fabric tuning) | n/a — peer track | `training-pipeline-engineer` (level 25) | Out of scope — eval runs on top of the platform |
| 18 | RAG engineering (chunking, embeddings, vector stores, rerankers) | n/a — peer track | `rag-engineer` (level 25) | mod-107 covers RAG *eval* methodology; system engineering linked out |
| 19 | LLM application engineering (prompting, agents, tool design, product integration) | n/a — peer track | `llm-application-developer` (level 25) | Out of scope — mentioned only as the subject under evaluation |
| 20 | Alignment-risk methodology (harm modelling, red-team data generation) | n/a — peer track | `ai-risk-engineer` (level 30) | mod-109 owns measurement; harm-model design and red-team data generation linked out |
| 21 | Deep ML/AI security (model extraction, eval-set exfiltration, supply-chain attacks on judges) | n/a — higher level | `ai-infra-security-learning` (level 35) | Surfaced as awareness in mod-102 and mod-109; depth owned upstream |
| 22 | Governance / compliance / policy / dataset licensing review depth | n/a — peer track | `ai-governance-analyst` (level 25) | Surfaced as awareness in mod-102 / mod-112; depth owned upstream |

## Posting evidence

<!-- needs-research: populate with the ≥25 in-window postings sampled next cycle. Use the same table shape as `ai-infra-agentic-ai-engineer-learning/JOB_REQUIREMENTS.md`. -->

No postings were sampled this cycle. See the **Status** section above for the reason. The next autonomous research cycle should fan out across:

- ATS aggregators: `boards.greenhouse.io`, `jobs.lever.co`, `jobs.ashbyhq.com`, `app.workable.com`, `myworkdayjobs.com`
- Frontier-lab employer ATS roots: OpenAI, Anthropic, Google DeepMind, Meta GenAI, xAI, Mistral, Cohere, Reka, Adept, Inflection, Character.AI, AI21, Apple AI/ML, Amazon AGI, Microsoft AI, Salesforce AI, NVIDIA, Snowflake Cortex
- AI-safety employers: UK AI Safety Institute, US AI Safety Institute, METR, Apollo Research, Redwood Research, MATS, Conjecture
- Open-weight / model-host employers: Hugging Face, Databricks (Mosaic), Snowflake (Cortex), NVIDIA NIM, Predibase, Fireworks, OctoAI, Modal
- Eval-tooling / observability employers: Arize AI (Phoenix), Trulens, Patronus, Galileo, Langfuse, Confident-AI / DeepEval, Lakera, Robust Intelligence, Weights & Biases (Weave), Snorkel, Argilla, Cleanlab
- Annotator / human-eval vendors paired with eval roles: Scale AI, Surge, Mercor, Prolific, Toloka
- Enterprise post-training / quality roles at: Salesforce AI Research, Bloomberg AI, JPMorgan AI Research, IBM Research, ServiceNow AI Research, Adobe Sensei
- Hyperscaler managed-eval product teams: Vertex AI Eval (Google), Azure AI Studio Eval (Microsoft), AWS Bedrock Model Evaluation
- Academic-spinout postings: Stanford CRFM affiliates, Berkeley BAIR affiliates, MIT-IBM Watson AI

For each posting, capture employer, exact title, URL, date_observed, date_posted (or `estimated:2026-MM`), location, 5–10 verbatim requirement bullets, 2–6 preferred-qualification bullets, salary range when published, and one short representative quote. Filter out Senior / Staff / Principal modifiers (those will inherit this packet), generic ML / Data Scientist / Research Scientist / AI-Application / Trust-&-Safety / Policy-Analyst titles (owned by peer tracks), and pure infra / platform titles (owned by the training-pipeline / ML platform tracks).

## Ownership map — quick reference for next cycle

When backfilling postings, use this ownership decision to keep the curriculum from drifting into peer territory:

- **Model Evaluation Engineer (this track, level 30)** owns evaluation engineering end-to-end at depth: validity & statistical methodology → benchmark engineering → harnesses across modalities (classical, LLM, multimodal, agentic) → human / LLM-judge calibration → safety & red-team measurement → production regression detection → eval platform engineering → release-gate systems design. Cross-modality (not LLM-only) and platform-altitude are the differentiators against peers.
- **ML Engineer** (level 20) owns the build-altitude practitioner workflow that this curriculum assumes (Python/PyTorch fluency, sklearn-style classical eval, FastAPI/Docker packaging, MLflow tracking).
- **AI Eval Engineer** (level 25, AI Engineering family) is the **peer** that owns build-altitude LLM/agent eval-harness work inside product pipelines (trajectory eval, eval-gated CI for prompts and chains, eval dashboards for app engineers).
- **AI Evaluation Engineer** (level 25, Governance family) is the **peer** that owns release-assurance / governance shape (audit trails, regulator-facing documentation, third-party evaluator interface).
- **Fine-Tuning Engineer** (level 30) is the **peer specialist** that produces the models this curriculum evaluates.
- **AI Risk Engineer** (level 30) is the **peer specialist** that designs harm models and generates red-team data; this curriculum *measures* safety properties from that input.
- **Senior ML Engineer** (level 30, peer ladder) is the *generalist* peer at the same level — overlaps in eval vocabulary but does not own the depth.
- **Staff ML Engineer** (level 40) and higher inherit / link to this packet for evaluation depth, but add architectural and cross-team scope.

## Differentiation versus the three peer "eval" tracks

The repository carries three eval-related tracks that share vocabulary. Posting research must disambiguate carefully:

| Track | Family | Owns |
|---|---|---|
| `model-evaluation-engineer` (this, level 30) | ML Engineering | Benchmark engineering depth, statistical methodology depth, multi-modality (classical / LLM / multimodal / agent), eval platform architecture |
| `ai-eval-engineer` (level 25) | AI Engineering | Build-altitude LLM/agent harness practitioner work inside product apps (trajectory eval, eval-gated CI, eval dashboards) |
| `ai-evaluation-engineer` (level 25) | Governance | Release-assurance / governance shape — audit trails, regulator-facing docs, third-party evaluator interface |

If a posting emphasises **benchmark construction, statistical rigour, eval platform internals, or cross-modality measurement at depth**, route it here. If it emphasises **product-side eval pipelines, prompt CI, or app-team enablement**, route it to `ai-eval-engineer`. If it emphasises **audit trails, regulator interface, or release-governance evidence**, route it to `ai-evaluation-engineer`.
