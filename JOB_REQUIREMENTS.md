# Job Requirements — Model Evaluation Engineer

**Role level:** 30 (deep specialist — peer to Senior ML Engineer on the ladder)
**Track:** `model-evaluation-engineer-learning`
**Research window:** 2026-05-05 → 2026-08-03 (last 90 days)
**Today:** 2026-08-03
**Postings sampled this cycle:** 30

This file documents the requirements catalog for the Model Evaluation Engineer curriculum. Raw normalized posting data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the current curriculum plan lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json); this cycle's proposed delta (empty by design — see the rationale in [Delta this cycle](#delta-this-cycle)) lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Sampling summary

Five parallel research agents fanned out across five employer segments to collect ≥25 live postings within the 90-day window (2026-05-05 → 2026-08-03):

| # | Segment | Postings | Employers this cycle |
|---|---|---|---|
| A | Frontier labs | 7 | Anthropic (×2), OpenAI (×2), Apple, NVIDIA, Meta MSL |
| B | Eval-tooling / annotator vendors | 6 | Scale AI, Nous Research, Ellipsis Health, Braintrust, Triomics, Surge AI |
| C | AI-safety institutes & frontier-safety research | 5 | Apollo Research, UK AISI, METR, Epoch AI (×2) |
| D | Hyperscalers & enterprise AI teams | 7 | Google (×4), Microsoft AI, ServiceNow (Moveworks), Salesforce |
| E | Aggregator/ATS tail | 5 | Firecrawl, White Circle, AGI Inc., Innodata, Glean |

Filters applied per the packet spec (see [`.aicg/job-requirements.json`](.aicg/job-requirements.json) → `research_status.sampling_strategy` for the exact wording):

- Excluded `Senior` / `Staff` / `Principal` modifiers (they inherit from this packet) — dropped one Tekion "Staff SDET AI Evaluation" posting on this rule.
- Excluded generic ML Engineer / Data Scientist / Research Scientist / AI Application Engineer / Trust & Safety / Policy Analyst titles owned by peer tracks.
- Excluded pure infra / platform / MLOps roles owned by training-pipeline / ml-platform tracks.
- Excluded postings observably outside 2026-05-05 → 2026-08-03 — dropped one Variance "RE Evals" posting dated 2026-03-31.

## Delta this cycle

**Proposed net-new modules:** 0
**Proposed net-new exercises:** 0
**Proposed net-new projects:** 0

Per the packet's continuity-bias rules, an addition requires **all three** of: (a) ≥3 distinct in-window postings citing a requirement the existing curriculum does NOT cover, (b) ≥30% posting frequency, (c) no incremental extension of an existing module/exercise/project can cover it. No candidate this cycle satisfies all three. See [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) for the empty-arrays proposal with rationale, and see [Emerging patterns below threshold](#emerging-patterns-below-threshold) for candidates the next cycle should keep watching.

## Requirement themes → curriculum ownership (with evidence)

`Freq` is `evidence-cited-postings / 30`. Postings are cited by their `posting_id` from [`.aicg/job-requirements.json`](.aicg/job-requirements.json) → `postings[]`; open that file to see verbatim requirement bullets and source URLs.

| # | Theme | Freq | Owner role | Coverage | Representative posting evidence |
|---|---|---|---|---|---|
| 1 | Evaluation methodology: validity, point estimates with CIs, paired tests, FDR control | 12/30 (0.40) | `model-evaluation-engineer` (this) | [`mod-101-evaluation-foundations`](lessons/mod-101-evaluation-foundations) | anthropic-mee-01/02 "Background in statistics and experimental design"; scale-mee-01 "statistical analysis foundation, with experience developing data-driven methods for assessing model quality"; nvidia-mee-01 "statistically sound comparisons for ML systems"; innodata-mee-01 "Strong foundation in experimental design, statistical analysis" |
| 2 | Benchmark engineering: sourcing, labelling with IAA, decontamination, versioning, canary sets | 17/30 (0.57) | `model-evaluation-engineer` | [`mod-102-benchmark-engineering`](lessons/mod-102-benchmark-engineering) | meta-mee-01 "Curate and integrate publicly available and internal benchmarks"; epoch-mee-01 "design and develop brand new benchmarks... primarily using the Inspect library"; firecrawl-mee-01 "Generate the datasets, adversarial cases, and golden sets that make measurements trustworthy"; whitecircle-mee-01 "Build and maintain internal suite of benchmarks" |
| 3 | Classical ML eval depth: calibration, slicing, fairness, threshold selection | 2/30 (0.07) | `model-evaluation-engineer` | [`mod-103-classical-ml-eval-depth`](lessons/mod-103-classical-ml-eval-depth) | google-mee-02 "Evaluate model behavior across languages, locales, and different hardware"; google-mee-03 "quantized model evaluation and mobile hardware benchmarking" |
| 4 | LLM benchmark harnesses: lm-eval-harness, HELM, OpenAI evals, Inspect, prompt-format sensitivity | 7/30 (0.23) | `model-evaluation-engineer` | [`mod-104-llm-benchmark-harnesses`](lessons/mod-104-llm-benchmark-harnesses) | apollo-mee-01 "Inspect evaluation framework"; aisi-mee-01 "Add a feature to one of our 'sandbox plugins' for Inspect"; epoch-mee-01/02 "Experience with the Inspect evaluation library"; openai-mee-02 "Track record contributing to open-sourced evaluations (e.g. SWE-bench Verified, MLE-bench, PaperBench)" |
| 5 | LLM-as-judge: rubric design, position / length / self-preference bias, calibration to humans, Arena-style ELO | 6/30 (0.20) | `model-evaluation-engineer` | [`mod-105-llm-as-judge-platforms`](lessons/mod-105-llm-as-judge-platforms) | scale-mee-01 "designing, building, or deploying LLM-as-a-Judge frameworks"; braintrust-mee-01 "Experience with LLM-as-a-judge scoring frameworks"; nous-mee-01 "developing LLM-as-judge pipelines"; apple-mee-01 "offline eval, human eval, A/B, or model-graded approaches" |
| 6 | Human evaluation: annotator workflows, IAA, gold-set rotation, vendor choice | 4/30 (0.13) | `model-evaluation-engineer` | [`mod-106-human-evaluation`](lessons/mod-106-human-evaluation) | scale-mee-01 "collaborating with operations or external teams to define high-quality human annotator guidelines"; surge-mee-01 "defining rubrics or reward signals for coding evaluation"; innodata-mee-01 "evaluate and compare human and automated evaluation methods, including tradeoffs in cost, reliability, validity, and scalability" |
| 7 | Generative & multimodal eval: pass@k code, math, RAG (RAGAS / TruLens), multimodal | 7/30 (0.23) | `model-evaluation-engineer` | [`mod-107-generative-and-multimodal-eval`](lessons/mod-107-generative-and-multimodal-eval) | meta-mee-01 "benchmarks or building RL environments for frontier LLMs across text, vision, or audio"; google-mee-04 "evaluation methodologies specific to generative AI in health-related applications"; innodata-mee-01 "long-context, cross-modal, and dynamic multi-turn evaluations"; servicenow-mee-01 "large language models, retrieval augmented generation (RAG), and agents" |
| 8 | Agent & tool eval: trajectory scoring, SWE-bench, WebArena / GAIA / AgentBench, METR tasks, Inspect agent harness | 17/30 (0.57) | `model-evaluation-engineer` | [`mod-108-agent-and-tool-eval`](lessons/mod-108-agent-and-tool-eval) | metr-mee-01 "integrating models into our agent scaffolds, running them on our infrastructure and checking the results carefully"; nous-mee-01 "agentic evaluation harnesses, tool-use, and multi-step reasoning tasks"; apple-mee-01 title = "Software Engineer, Agentic Evaluation"; whitecircle-mee-01 "single/multi-turn content and agentic guardrails"; glean-mee-01 "agent observability infrastructure, including trace enrichment, durable telemetry pipelines" |
| 9 | Safety / red-team eval: refusal, jailbreak resistance (HarmBench), dangerous-capability evals, prompt-injection robustness, bias / toxicity | 10/30 (0.33) | `model-evaluation-engineer` | [`mod-109-safety-and-red-team-eval`](lessons/mod-109-safety-and-red-team-eval) | apollo-mee-01 "Conducting large-scale AI red-teaming exercises... alignment faking and scheming"; openai-mee-01 "First-hand experience in red-teaming systems"; salesforce-mee-01 "MITRE ATLAS and the OWASP Top 10 for LLMs"; nvidia-mee-01 "AI Safety and Security Engineering"; aisi-mee-01 "safe mechanisms to let models write and execute arbitrary code" |
| 10 | Production eval & regression: offline gates, shadow, A/B with CUPED, sequential testing, drift / judge drift, MLPerf | 11/30 (0.37) | `model-evaluation-engineer` | [`mod-110-production-eval-regression`](lessons/mod-110-production-eval-regression) | anthropic-mee-01/02 "make regressions impossible to miss... on-call or production-support capacity when training runs are live"; triomics-mee-01 "regression testing, release validation, and production impact analysis for clinical AI systems"; glean-mee-01 "regressions, tradeoffs, and launch readiness"; ellipsis-mee-01 "test automation frameworks, evaluation pipelines, or CI/CD-integrated testing systems" |
| 11 | Eval platform engineering: registry, multi-runner orchestration, eval data warehouse, CI integration, SLO | 18/30 (0.60) | `model-evaluation-engineer` | [`mod-111-eval-platform-engineering`](lessons/mod-111-eval-platform-engineering) + [`project-103-eval-platform-capstone`](projects/project-103-eval-platform-capstone) | microsoft-mee-01 "Design and build the evaluation infrastructure for generative AI on large-scale GPU clusters"; anthropic-mee-01/02 "Build and harden the distributed eval execution platform so hundreds of evals run reliably against checkpoints"; metr-mee-01 "Streamlining processes and building common infrastructure to scale our ability to continually run our most up-to-date evaluations"; glean-mee-01 "Design and build large-scale evaluation pipelines that measure assistant and agent quality" |
| 12 | Eval systems design & governance interface: release gates, model cards, NIST AI RMF / ISO 25059 / EU AI Act mapping, build-vs-buy | 6/30 (0.20) | `model-evaluation-engineer` | [`mod-112-eval-systems-design`](lessons/mod-112-eval-systems-design) | metr-mee-01 "designing useful graphs and writing up conclusions for different audiences (system cards, risk reports)"; apollo-mee-01 "Producing technical reports for partner AI laboratories"; triomics-mee-01 "Produce release-readiness reports for stakeholders" |
| 13 | PyTorch / sklearn-eval / FastAPI / Docker / experiment tracking fundamentals | n/a — prerequisite | `ml-engineer` (level 20) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught. Python fluency appears in ~all sampled postings but as a *prerequisite* not a curriculum topic. | — |
| 14 | Build-altitude LLM/agent eval-harness practitioner work in product pipelines | n/a — peer track | `ai-eval-engineer` (level 25, AI Engineering family) | Linked out; this curriculum stays at methodology / platform altitude | — |
| 15 | Release-assurance / governance-shaped evaluation (audit trails, regulator interface) | n/a — peer track | `ai-evaluation-engineer` (level 25, Governance family) | This curriculum produces the evidence; governance shape is owned upstream | — |
| 16 | Post-training (SFT / PEFT / RLHF / DPO) depth | n/a — peer track | `fine-tuning-engineer` (level 30) | Out of scope — we evaluate the outputs. innodata-mee-01's "LLM Evaluation & Post-Training" title splits along this boundary: retained here for the eval half, deferred for the post-training half. | — |
| 17 | Distributed-training PLATFORM engineering (multi-tenant schedulers, NCCL/fabric tuning) | n/a — peer track | `training-pipeline-engineer` (level 25) | Out of scope — eval runs on top of the platform. microsoft-mee-01's "evaluation infrastructure on large-scale GPU clusters" is retained here for the *eval-plane* platform; the *training-plane* platform stays with the training-pipeline track. | — |
| 18 | RAG engineering (chunking, embeddings, vector stores, rerankers) | n/a — peer track | `rag-engineer` (level 25) | mod-107 covers RAG *eval* methodology; RAG system engineering (servicenow-mee-01's primary skill) linked out | — |
| 19 | LLM application engineering (prompting, agents, tool design, product integration) | n/a — peer track | `llm-application-developer` (level 25) | Out of scope — mentioned only as the subject under evaluation | — |
| 20 | Alignment-risk methodology (harm modelling, red-team data generation) | n/a — peer track | `ai-risk-engineer` (level 30) | mod-109 owns measurement; harm-model design and red-team data generation linked out. apollo-mee-01's "alignment faking and scheming" is retained here for the detection/measurement side, deferred for the harm-modelling side. | — |
| 21 | Deep ML/AI security (model extraction, eval-set exfiltration, supply-chain attacks on judges) | n/a — higher level | `ai-infra-security-learning` (level 35) | Surfaced as awareness in mod-102 and mod-109; depth owned upstream. salesforce-mee-01 sits mostly here and is retained only for its eval-pipeline-construction requirement. | — |
| 22 | Governance / compliance / policy / dataset licensing review depth | n/a — peer track | `ai-governance-analyst` (level 25) | Surfaced as awareness in mod-102 / mod-112; depth owned upstream | — |

### Reading the evidence-frequency numbers

- **Denominator = 30** (postings sampled this cycle). The window closed on 2026-08-03; if a requirement is not cited by ≥30% (≥10/30) of postings, the packet does not authorise net-new curriculum content for it, even if the requirement is widely accepted as important. See [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) for the empty-arrays proposal.
- **Low-frequency but essential-foundation requirements** (mod-103 classical eval at 0.07; mod-106 human eval at 0.13) are retained at full depth because they underlie higher-frequency work. Postings sample the surface (what the JD asks about); the curriculum teaches the foundation and the surface. Do not demote without cross-referencing peer-track posting evidence.
- **Highest-frequency requirements this cycle**: eval platform engineering (0.60), benchmark engineering (0.57), agent & tool eval (0.57), evaluation methodology (0.40), production regression (0.37), safety/red-team (0.33).

## Posting evidence — headline table

Full verbatim requirement bullets, preferred bullets, salary bands, source URLs, and per-posting notes live in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) under `postings[]`. Summary here for orientation.

| posting_id | Employer | Title | Date posted | Location | Salary (as published) |
|---|---|---|---|---|---|
| anthropic-mee-01 | Anthropic | Research Engineer, Model Evaluations | 2026-05-05 | San Francisco, CA | $320K – $485K |
| anthropic-mee-02 | Anthropic | Research Engineer, Model Evaluations | est. 2026-07 | SF / NYC / Remote-Friendly | $500K – $850K |
| openai-mee-01 | OpenAI | Research Engineer, Frontier Evals & Environments | est. 2026-07 | San Francisco, CA | $205K – $380K |
| openai-mee-02 | OpenAI | Research Engineer, Frontier Evals & Environments – Finance | est. 2026-06 | San Francisco, CA | $200K – $370K |
| apple-mee-01 | Apple | Software Engineer, Agentic Evaluation | est. 2026-05 | Cupertino, CA | $147K – $272K base |
| nvidia-mee-01 | NVIDIA | Evaluation and ML Systems Engineer, AI Safety and Security Engineering | 2026-07-27 | Santa Clara, CA (remote-eligible) | Not published |
| meta-mee-01 | Meta MSL | Research Engineer, Evaluations | est. 2026-06 | Menlo Park, CA | Not published |
| scale-mee-01 | Scale AI | AI Research Engineer, Enterprise Evaluations | est. 2026-05 | SF / NYC / Seattle | $179K – $224K |
| nous-mee-01 | Nous Research | Machine Learning Engineer, Evals | est. 2026-07 | NYC (Remote) | Not published |
| ellipsis-mee-01 | Ellipsis Health | AI Evaluations Engineer – Healthcare | est. 2026-06 | Remote, US | $150K – $180K |
| braintrust-mee-01 | Braintrust | Eval Engineer | est. 2026-07 | SF or NYC | Not published |
| triomics-mee-01 | Triomics | ML Model Evaluation Engineer | 2026-07-03 | US/India hybrid | Not published |
| surge-mee-01 | Surge AI | Software Engineer, Coding Evaluation & Training Data | est. 2026-06 | Remote-friendly | Not published |
| apollo-mee-01 | Apollo Research | Research Scientist/Engineer (Evaluations) | 2026-07-26 | London, UK or SF | £100K – £200K (London) |
| aisi-mee-01 | UK AISI | Software Engineer, Core Technology Team | est. 2026-07 | London (multi-site UK) | £65K – £145K + 28.97% pension |
| metr-mee-01 | METR | Member of Technical Staff, Evaluation Execution | est. 2026-06 | Berkeley, CA | $285K – $503K |
| epoch-mee-01 | Epoch AI | Software Engineer, Benchmarking | est. 2026-06 | Remote (global) | $125K – $275K |
| epoch-mee-02 | Epoch AI | Researcher, Evaluations | est. 2026-06 | Remote (PT/GMT overlap) | $115K – $200K |
| google-mee-01 | Google | Software Engineer, AI Evaluations (Pixel/Android) | est. 2026-06 | United States | Not published on posting |
| google-mee-02 | Google | Software Engineer, AI Quality and Benchmarks | est. 2026-06 | United States | Not published on posting |
| google-mee-03 | Google | Software Engineer, On-device AI Model Evaluation | est. 2026-06 | United States | Not published on posting |
| google-mee-04 | Google Research / Health AI | Research Software Engineer, Generative AI Evaluations, Health AI | est. 2026-05 | United States | Not published on posting |
| microsoft-mee-01 | Microsoft AI (MAI Superintelligence Team) | Member of Technical Staff, Evaluations Engineer | 2026-06-02 | Multiple US locations | IC5 $142K–$304K; IC6 $165K–$331K |
| servicenow-mee-01 | ServiceNow (Moveworks) | ML Engineer, GAI Search Platform | est. 2026-07 | Mountain View, CA | $139K – $216K |
| salesforce-mee-01 | Salesforce (Trust / AI Research) | Adversarial AI & Research Engineer | est. 2026-06 | Multi-site US (Remote / SF) | $148K – $246K (SF/NYC) |
| firecrawl-mee-01 | Firecrawl | Research Engineer (Evals) | 2026-07-12 | SF (Hybrid) | $210K – $275K |
| whitecircle-mee-01 | White Circle | Research Engineer (Evals) | 2026-06-28 | Paris (Hybrid) / London | $120K – $250K |
| agi-inc-mee-01 | AGI, Inc. | Research Engineer, Evals | 2026-05-27 | San Francisco | Not published |
| innodata-mee-01 | Innodata Inc. | Applied Research Scientist, LLM Evaluation & Post-Training | est. 2026-06 | Remote, Canada | CAD $245K – $315K |
| glean-mee-01 | Glean | Software Engineer, Evals | est. 2026-07 | Bangalore, India | Not published on posting |

Two postings were considered but excluded per the sampling rules:

- `variance-mee-01` — Variance "Research Engineer, Evals" — posted 2026-03-31, 35 days outside the 90-day window.
- `tekion-mee-01` — Tekion "Staff Software Development Test Engineer – AI Evaluation" — carries the `Staff` modifier per rule (i) of the sampling strategy; that shape inherits from this packet at a higher level.

## Ownership map — quick reference

- **Model Evaluation Engineer (this track, level 30)** owns evaluation engineering end-to-end at depth: validity & statistical methodology → benchmark engineering → harnesses across modalities (classical, LLM, multimodal, agentic) → human / LLM-judge calibration → safety & red-team measurement → production regression detection → eval platform engineering → release-gate systems design. Cross-modality and platform-altitude remain the differentiators against peer tracks.
- **ML Engineer** (level 20) owns the build-altitude practitioner workflow that this curriculum assumes.
- **AI Eval Engineer** (level 25, AI Engineering family) is the **peer** that owns build-altitude LLM/agent eval-harness work inside product pipelines.
- **AI Evaluation Engineer** (level 25, Governance family) is the **peer** that owns release-assurance / governance shape (audit trails, regulator-facing documentation, third-party evaluator interface).
- **Fine-Tuning Engineer** (level 30) is the **peer specialist** that produces the models this curriculum evaluates.
- **AI Risk Engineer** (level 30) is the **peer specialist** that designs harm models and generates red-team data; this curriculum *measures* safety properties from that input.
- **Senior ML Engineer** (level 30, peer ladder) is the generalist peer at the same level.
- **Staff ML Engineer** (level 40) and higher inherit / link to this packet for evaluation depth.

## Differentiation versus the three peer "eval" tracks

| Track | Family | Owns | Posting shape |
|---|---|---|---|
| `model-evaluation-engineer` (this, level 30) | ML Engineering | Benchmark engineering depth, statistical methodology, multi-modality (classical / LLM / multimodal / agent), eval platform architecture | The 30 postings in this packet |
| `ai-eval-engineer` (level 25) | AI Engineering | Build-altitude LLM/agent harness practitioner work inside product apps (trajectory eval, eval-gated CI, eval dashboards for app teams) | App-embedded eval postings |
| `ai-evaluation-engineer` (level 25) | Governance | Release-assurance / governance shape — audit trails, regulator-facing docs, third-party evaluator interface | Regulator-facing / QMS postings |

If a posting emphasises **benchmark construction, statistical rigour, eval platform internals, or cross-modality measurement at depth**, route it here. If it emphasises **product-side eval pipelines, prompt CI, or app-team enablement**, route it to `ai-eval-engineer`. If it emphasises **audit trails, regulator interface, or release-governance evidence**, route it to `ai-evaluation-engineer`.

## Emerging patterns below threshold

These patterns appear in the sample but fall below the 30% frequency threshold that would authorise net-new curriculum content this cycle. They are recorded here so the next cycle can pick them up if the frequency rises; each links to the adjacent existing coverage that would extend to absorb them if promoted.

1. **RL environment authoring for capability elicitation** — the eval doubles as an RL training environment; the eval engineer authors reward-generating environments (not just scoring rubrics) that steer the training run. **Freq: 3/30 (0.10)**. Postings: `openai-mee-01`, `openai-mee-02`, `meta-mee-01`. Adjacent coverage: [`mod-108-agent-and-tool-eval`](lessons/mod-108-agent-and-tool-eval) already covers sandboxed trajectory scoring and partial-credit rubrics; environment authoring would be an exercise extension there, not a new module.
2. **First-class eval dashboards and visualization** — dashboards researchers *and* leadership trust for training-run decisions. **Freq: 7/30 (0.23)**. Postings: `anthropic-mee-01`, `anthropic-mee-02`, `firecrawl-mee-01`, `agi-inc-mee-01`, `glean-mee-01`, `google-mee-02`, `metr-mee-01`. Adjacent coverage: [`mod-111-eval-platform-engineering`](lessons/mod-111-eval-platform-engineering) already covers the data-warehouse-and-orchestration substrate; dashboarding would be a UI/UX-of-eval extension exercise.
3. **On-device / mobile / OEM-hardware ML eval** — quantized-model quality regression, latency-vs-quality on device, cross-hardware benchmark reproducibility. **Freq: 4/30 (0.13)**. Postings: `apple-mee-01`, `google-mee-01`, `google-mee-03`, `agi-inc-mee-01`. Adjacent coverage: [`mod-110-production-eval-regression`](lessons/mod-110-production-eval-regression) already covers MLPerf-style serving benchmarks; an on-device exercise would extend that module.
4. **Agent trace-level observability** — span enrichment, durable telemetry, debugging workflows for agent behavior. **Freq: 4/30 (0.13)**. Postings: `glean-mee-01`, `ellipsis-mee-01`, `nous-mee-01`, `anthropic-mee-01`. Adjacent coverage: mod-110 already integrates Arize Phoenix / Langfuse / W&B Weave. Not a new-content signal this cycle.
5. **Domain-specialised eval (finance, healthcare, clinical)** — eval expertise *in* the vertical rather than generic LLM eval alone. **Freq: 4/30 (0.13)**. Postings: `openai-mee-02` (finance), `google-mee-04` (health), `ellipsis-mee-01` (health), `triomics-mee-01` (clinical). Adjacent coverage: [`project-101-benchmark-engineering-capstone`](projects/project-101-benchmark-engineering-capstone) already lists a domain menu (medical Q&A, legal summarisation, code review, customer support, analyst data Q&A) — no change needed unless the frequency rises.

## What the next cycle should check

- Rerun the same 5-segment fan-out. Watch whether **RL environment authoring** and **first-class eval dashboards** clear the 30% threshold; if so, add exercises to `mod-108` and `mod-111` respectively (not new modules).
- Recheck **classical ML eval depth** (0.07 this cycle) against fine-tuning-engineer and senior-ml-engineer posting evidence before considering demotion — this module's low direct citation is expected in an LLM-dominated sample but may still be foundational.
- Widen the geographic coverage — this cycle skewed heavily US/UK. Add EMEA/APAC enterprise-eval postings in the next fan-out.
- Recheck whether hyperscaler eval hiring (Amazon Bedrock, Databricks Mosaic, Bloomberg AI, JPMorgan AI Research) has any *non*-Senior/Staff/Principal shape appearing — they contributed zero postings this cycle because their published roles carry filtered-out modifiers.
