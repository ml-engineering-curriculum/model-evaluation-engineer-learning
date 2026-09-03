# Job Requirements — Model Evaluation Engineer

**Role level:** 30 (deep specialist — peer to Senior ML Engineer on the ladder)
**Track:** `model-evaluation-engineer-learning`
**Research window:** 2026-06-05 → 2026-09-03 (last 90 days)
**Today:** 2026-09-03
**Postings sampled this cycle:** 31

This file documents the requirements catalog for the Model Evaluation Engineer curriculum. Raw normalized posting data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the current curriculum plan lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json); this cycle's proposed delta (empty by design — see the rationale in [Delta this cycle](#delta-this-cycle)) lives in [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json).

## Sampling summary

Five parallel research agents fanned out across five employer segments to collect ≥25 live postings within the 90-day window (2026-06-05 → 2026-09-03):

| # | Segment | Postings | Employers this cycle |
|---|---|---|---|
| A | Frontier labs | 8 | Anthropic, OpenAI (×2), Apple, NVIDIA, Meta MSL, Google DeepMind, xAI (dropped: out of window) |
| B | Eval-tooling / annotator vendors | 5 | Scale AI, Nous Research, Ellipsis Health, Mercor, Surge AI |
| C | AI-safety institutes & frontier-safety research | 7 | Apollo Research (×2 — split this cycle), UK AISI, METR, Epoch AI (×2), CAIS |
| D | Hyperscalers & enterprise AI teams | 7 | Google (×4), Microsoft AI (health), ServiceNow (Moveworks), Salesforce |
| E | Aggregator/ATS tail | 5 | Firecrawl, White Circle, Innodata, Glean, Triomics |

**Equivalent titles counted per the packet's fragmented-role rule:** `Model Evaluation Engineer`, `ML Evaluation Engineer`, `Benchmark Engineer`, `Eval Engineer`, plus their common variants (`Research Engineer, Evaluations / Evals / Frontier Evals`, `Software Engineer, AI Evaluations / Agentic Evaluation / AI Quality and Benchmarks / On-device AI Model Evaluation`, `Machine Learning Engineer, Evals`, `Member of Technical Staff, Evaluations`, `Applied Research Scientist, LLM Evaluation`). Every posting sampled here maps to one of those shapes.

Filters applied per the packet spec (see [`.aicg/job-requirements.json`](.aicg/job-requirements.json) → `research_status.sampling_strategy` for the exact wording):

- Excluded `Senior` / `Staff` / `Principal` / `Lead` modifiers (they inherit from this packet) — dropped Sentry "Senior SWE, AI Evals," Artificial Analysis "Senior AI/ML Engineer," P-1 AI "AI Evals Technical Lead," Bloomberg "Senior SWE AI App Enablement & Observability" on this rule.
- Excluded generic ML Engineer / Data Scientist / Research Scientist / AI Application Engineer / Trust & Safety / Policy Analyst titles owned by peer tracks — dropped ByteDance "AI Product Manager (Evaluation)" as a PM (not engineering) title.
- Excluded pure infra / platform / MLOps roles owned by training-pipeline / ml-platform tracks.
- Excluded postings observably outside 2026-06-05 → 2026-09-03 — dropped Anthropic RE Model Evaluations (2026-05-05, 31 days outside; equivalent Anthropic role retained via `anthropic-mee-02` at the newer 2026-07-10 date), AGI Inc. (2026-05-27, 9 days outside), Mercor "SWE AI Data & Evaluation" (2026-05-29, 7 days outside; equivalent Mercor role retained via `mercor-mee-01` at the newer estimated-2026-08 date), xAI MTS Model Evaluation (2026-03-18, 79 days outside), Bloomberg (2026-04-28, also failed the seniority filter).
- Removed Braintrust Eval Engineer (`braintrust-mee-01` from prior cycle) — canonical URL now returns 404 and no equivalent Braintrust IC-level eval-engineer opening is currently posted.

## Delta this cycle

**Proposed net-new modules:** 0
**Proposed net-new exercises:** 0
**Proposed net-new projects:** 0

Per the packet's continuity-bias rules, an addition requires **all three** of: (a) ≥3 distinct in-window postings citing a requirement the existing curriculum does NOT cover, (b) ≥30% posting frequency, (c) no incremental extension of an existing module/exercise/project can cover it. No candidate this cycle satisfies all three. See [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) for the empty-arrays proposal with rationale, and see [Emerging patterns below threshold](#emerging-patterns-below-threshold) for candidates the next cycle should keep watching.

## What changed since the 2026-08 cycle

The market moved incrementally in ways that reinforce — not destabilize — the existing 12-module curriculum:

- **`req-eval-platform-engineering`** is even more dominant than last cycle (0.68 vs 0.60). Every net-new employer (Google DeepMind, CAIS, Mercor, Apollo Evals-SWE) cites it. Already fully covered by [`mod-111`](lessons/mod-111-eval-platform-engineering) + [`project-103`](projects/project-103-eval-platform-capstone).
- **`req-agent-tool-eval`** climbs to 0.61 (was 0.57). Nous now names the full agent-benchmark taxonomy explicitly (GAIA, τ-Bench, MINT, SWE-bench, WebShop, ALFWorld, AgentBench) in a single JD — the strongest single-posting evidence for [`mod-108`](lessons/mod-108-agent-and-tool-eval). Microsoft's Mayo Clinic role and Google's AI Quality role both cite "agentic/trajectory evaluation" as first-class product-team work.
- **`req-safety-redteam-eval`** climbs to 0.39 (was 0.33). Google DeepMind's Frontier Safety Risk Assessment role adds a third public frontier-safety framework (Anthropic RSP + OpenAI Preparedness + DeepMind FSF) to the mapping [`mod-109`](lessons/mod-109-safety-and-red-team-eval) already teaches; the DeepMind FSF is now added to the module's authoritative-reference set. Apollo's new Evals-SWE lane carries the scheming/alignment-faking framing forward alongside the research-heavy lane.
- **Inspect harness dominance widens**: 5 of 31 postings now name Inspect explicitly (Apollo x2, AISI, Epoch x2). Apollo's Evals-SWE posting says outright, "Apollo Research has recently switched to Inspect as our primary evals framework; our entire stack uses Python." [`mod-104`](lessons/mod-104-llm-benchmark-harnesses) should keep Inspect front-and-center. Nous introduces open-frontier alternatives ("Harbor, Nemo Evaluator") by name — worth mentioning as adjacent options in mod-104 without displacing Inspect.
- **New signal — "failure analysis" as first-class title concept**: Mercor's new posting has "Failure Analysis" in its title; Nous asks candidates to "analyze model failures and turn them into measurable signals"; Anthropic asks eval engineers to "debug anomalous eval results mid-training-run." 3/31 = 0.10, well below threshold. Logged under [Emerging patterns below threshold](#emerging-patterns-below-threshold) as a new watch item. Adjacent coverage already exists inside mod-108 (partial-credit / trajectory diagnosis) and mod-110 (regression triage).
- **New signal — Apollo splits its evals hire into research-heavy and SWE-execution lanes** (mirroring METR's Execution vs Research split). This is evidence that eval-execution SWE is becoming a distinct job shape. The existing curriculum's methodology + platform + benchmark modules already prepare graduates for both lanes — no content change needed.
- **New signal — RL environment authoring hardens at frontier labs**: OpenAI now lists "Create ambitious RL environments" as bullet #1; Meta MSL lists "Develop and implement evaluation environments"; Surge and Mercor add adjacent "RL environments" framing. Frequency up from 0.10 to 0.19 — still below threshold. Adjacent coverage in mod-108; watch next cycle.
- **New signal — domain-specialized eval strengthens**: Microsoft's new Mayo Clinic-embed evals role adds a fourth healthcare/clinical citation and the second frontier-lab-scale employer (after Google Health AI) in the health/clinical vertical. Frequency 0.16 (was 0.13), still below threshold. Adjacent coverage in project-101's domain menu.
- **`req-eval-methodology-foundations`** climbs to 0.48 (was 0.40), driven by Nous's refreshed JD (which now explicitly cites Cohen's κ, confidence intervals on metrics, non-determinism sampling, and psychometrics/measurement theory) plus CAIS and DeepMind adding "empirical research" framing. [`mod-101`](lessons/mod-101-evaluation-foundations) is already the correct home for all of these; no change needed.

Nothing in the above list clears the packet's ≥30% + uncovered + non-extendable test. Consequently the delta stays empty — this is the intended outcome, not a research failure.

## Requirement themes → curriculum ownership (with evidence)

`Freq` is `evidence-cited-postings / 31`. Postings are cited by their `posting_id` from [`.aicg/job-requirements.json`](.aicg/job-requirements.json) → `postings[]`; open that file to see verbatim requirement bullets and source URLs.

| # | Theme | Freq | Owner role | Coverage | Representative posting evidence |
|---|---|---|---|---|---|
| 1 | Evaluation methodology: validity, point estimates with CIs, paired tests, FDR control | 15/31 (0.48) | `model-evaluation-engineer` (this) | [`mod-101-evaluation-foundations`](lessons/mod-101-evaluation-foundations) | nous-mee-01 "why LLM-as-judge needs calibration; what Cohen's κ measures; how to think about confidence intervals on a metric; how you sample, how many runs, how you report variance"; deepmind-mee-01 "Adept at generating ideas and designing experiments"; cais-mee-01 "empirical research in AI or ML, particularly in AI safety-relevant areas (e.g. adversarial robustness, calibration, benchmarking)"; scale-mee-01 "statistical analysis foundation, with experience developing data-driven methods for assessing model quality"; nvidia-mee-01 "statistically sound comparisons for ML systems" |
| 2 | Benchmark engineering: sourcing, labelling with IAA, decontamination, versioning, canary sets | 18/31 (0.58) | `model-evaluation-engineer` | [`mod-102-benchmark-engineering`](lessons/mod-102-benchmark-engineering) | meta-mee-01 "Curate and integrate publicly available and internal benchmarks"; mercor-mee-01 "Designing, implementing, and maintaining benchmarks and metrics for tool use, agentic behavior, and real-world reasoning"; epoch-mee-01 "design and develop brand new benchmarks... primarily using the Inspect library"; firecrawl-mee-01 "Generate the datasets, adversarial cases, and golden sets that make measurements trustworthy"; cais-mee-01 "Build and maintain datasets and benchmarks" |
| 3 | Classical ML eval depth: calibration, slicing, fairness, threshold selection | 3/31 (0.10) | `model-evaluation-engineer` | [`mod-103-classical-ml-eval-depth`](lessons/mod-103-classical-ml-eval-depth) | google-mee-02 "Evaluate model behavior across languages, locales, and different hardware"; google-mee-03 "quantized model evaluation and mobile hardware benchmarking"; cais-mee-01 "adversarial robustness, calibration, benchmarking" |
| 4 | LLM benchmark harnesses: lm-eval-harness, HELM, OpenAI evals, Inspect, prompt-format sensitivity | 8/31 (0.26) | `model-evaluation-engineer` | [`mod-104-llm-benchmark-harnesses`](lessons/mod-104-llm-benchmark-harnesses) | apollo-mee-02 "Apollo Research has recently switched to Inspect as our primary evals framework; our entire stack uses Python"; aisi-mee-01 "Add a feature to one of our 'sandbox plugins' for Inspect"; epoch-mee-01/02 "Experience with the Inspect evaluation library"; nous-mee-01 "at least one LLM evaluation framework (Harbor, Nemo Evaluator, etc.)" |
| 5 | LLM-as-judge: rubric design, position / length / self-preference bias, calibration to humans, Arena-style ELO | 8/31 (0.26) | `model-evaluation-engineer` | [`mod-105-llm-as-judge-platforms`](lessons/mod-105-llm-as-judge-platforms) | scale-mee-01 "designing, building, or deploying LLM-as-a-Judge frameworks"; microsoft-mee-01 "LLM-as-judge design, user simulation, and agentic/trajectory evaluation"; firecrawl-mee-01 "validate judges"; mercor-mee-01 "build rubrics and scorers"; nous-mee-01 "developing LLM-as-judge pipelines" |
| 6 | Human evaluation: annotator workflows, IAA, gold-set rotation, vendor choice | 4/31 (0.13) | `model-evaluation-engineer` | [`mod-106-human-evaluation`](lessons/mod-106-human-evaluation) | scale-mee-01 "collaborating with operations or external teams to define high-quality human annotator guidelines"; surge-mee-01 "defining rubrics or reward signals for coding evaluation"; innodata-mee-01 "evaluate and compare human and automated evaluation methods" |
| 7 | Generative & multimodal eval: pass@k code, math, RAG (RAGAS / TruLens), multimodal | 8/31 (0.26) | `model-evaluation-engineer` | [`mod-107-generative-and-multimodal-eval`](lessons/mod-107-generative-and-multimodal-eval) | meta-mee-01 "benchmarks or building RL environments for frontier LLMs across text, vision, or audio"; google-mee-04 "evaluation methodologies specific to generative AI in health-related applications"; microsoft-mee-01 "training/evaluation of LLMs, including LLM-as-judge design, user simulation, and agentic/trajectory evaluation"; servicenow-mee-01 "large language models, retrieval augmented generation (RAG), and agents" |
| 8 | Agent & tool eval: trajectory scoring, SWE-bench, WebArena / GAIA / AgentBench / τ-Bench / MINT / ALFWorld / WebShop, METR tasks, Inspect agent harness | 19/31 (0.61) | `model-evaluation-engineer` | [`mod-108-agent-and-tool-eval`](lessons/mod-108-agent-and-tool-eval) | nous-mee-01 "you know at least two agent benchmarks (GAIA, AgentBench, τ-Bench, MINT, SWE-bench, WebShop, ALFWorld) and a limitation of each"; metr-mee-01 "integrating models into our agent scaffolds, running them on our infrastructure and checking the results carefully"; microsoft-mee-01 "agentic/trajectory evaluation"; apple-mee-01 title = "Software Engineer, Agentic Evaluation"; glean-mee-01 "agent observability infrastructure, including trace enrichment, durable telemetry pipelines" |
| 9 | Safety / red-team eval: refusal, jailbreak resistance (HarmBench), dangerous-capability evals, prompt-injection robustness, bias / toxicity | 12/31 (0.39) | `model-evaluation-engineer` | [`mod-109-safety-and-red-team-eval`](lessons/mod-109-safety-and-red-team-eval) | deepmind-mee-01 "Identifying new risk pathways within current areas (loss of control, ML R&D, cyber, CBRN, harmful manipulation)"; apollo-mee-01/02 "Conducting large-scale AI red-teaming exercises... alignment faking and scheming"; cais-mee-01 "adversarial robustness, calibration, benchmarking"; aisi-mee-01 "safe mechanisms to let models write and execute arbitrary code"; salesforce-mee-01 "MITRE ATLAS and the OWASP Top 10 for LLMs" |
| 10 | Production eval & regression: offline gates, shadow, A/B with CUPED, sequential testing, drift / judge drift, MLPerf | 11/31 (0.35) | `model-evaluation-engineer` | [`mod-110-production-eval-regression`](lessons/mod-110-production-eval-regression) | anthropic-mee-02 "make regressions impossible to miss... on-call or production-support capacity when training runs are live"; microsoft-mee-01 "defining what 'good' looks like for a Mayo Clinic agent starting from an explicit specification of intended behavior"; triomics-mee-01 "regression testing, release validation, and production impact analysis for clinical AI systems"; glean-mee-01 "regressions, tradeoffs, and launch readiness"; ellipsis-mee-01 "test automation frameworks, evaluation pipelines, or CI/CD-integrated testing systems"; mercor-mee-01 "scalability, quality, and operational reliability simultaneously" |
| 11 | Eval platform engineering: registry, multi-runner orchestration, eval data warehouse, CI integration, SLO | 21/31 (0.68) | `model-evaluation-engineer` | [`mod-111-eval-platform-engineering`](lessons/mod-111-eval-platform-engineering) + [`project-103-eval-platform-capstone`](projects/project-103-eval-platform-capstone) | anthropic-mee-02 "Build and harden the distributed eval execution platform so hundreds of evals run reliably against checkpoints"; microsoft-mee-01 (health) "Design and build the evaluation infrastructure... defining what 'good' looks like for a Mayo Clinic agent"; apollo-mee-02 "Building infrastructure that supports scalable AI evaluations"; metr-mee-01 "Streamlining processes and building common infrastructure to scale our ability to continually run our most up-to-date evaluations"; mercor-mee-01 "Systems thinking: ability to design for scalability, quality, and operational reliability simultaneously"; glean-mee-01 "Design and build large-scale evaluation pipelines that measure assistant and agent quality" |
| 12 | Eval systems design & governance interface: release gates, model cards, NIST AI RMF / ISO 25059 / EU AI Act / RSP / Preparedness / DeepMind FSF mapping, build-vs-buy | 8/31 (0.26) | `model-evaluation-engineer` | [`mod-112-eval-systems-design`](lessons/mod-112-eval-systems-design) | deepmind-mee-01 "de-risk model launches by researching and implementing defenses against high-stakes frontier safety risks"; microsoft-mee-01 "HIPAA, PHI handling, and/or lifecycle regulatory approaches to clinical AI"; metr-mee-01 "designing useful graphs and writing up conclusions for different audiences (system cards, risk reports)"; apollo-mee-01/02 "Producing technical reports for partner AI laboratories"; triomics-mee-01 "Produce release-readiness reports for stakeholders" |
| 13 | PyTorch / sklearn-eval / FastAPI / Docker / experiment tracking fundamentals | n/a — prerequisite | `ml-engineer` (level 20) | Listed in [`PREREQUISITES.md`](PREREQUISITES.md); not re-taught. Python fluency appears in ~all sampled postings but as a *prerequisite* not a curriculum topic. | — |
| 14 | Build-altitude LLM/agent eval-harness practitioner work in product pipelines | n/a — peer track | `ai-eval-engineer` (level 25, AI Engineering family) | Linked out; this curriculum stays at methodology / platform altitude | — |
| 15 | Release-assurance / governance-shaped evaluation (audit trails, regulator interface) | n/a — peer track | `ai-evaluation-engineer` (level 25, Governance family) | This curriculum produces the evidence; governance shape is owned upstream | — |
| 16 | Post-training (SFT / PEFT / RLHF / DPO) depth | n/a — peer track | `fine-tuning-engineer` (level 30) | Out of scope — we evaluate the outputs. innodata-mee-01's "LLM Evaluation & Post-Training" title splits along this boundary: retained here for the eval half, deferred for the post-training half. | — |
| 17 | Distributed-training PLATFORM engineering (multi-tenant schedulers, NCCL/fabric tuning) | n/a — peer track | `training-pipeline-engineer` (level 25) | Out of scope — eval runs on top of the platform. The prior microsoft-mee-01 (MAI Superintelligence Team Evaluations Engineer, now 404) framed "evaluation infrastructure on large-scale GPU clusters" as an eval-plane responsibility; the replacement microsoft-mee-01 (AI Evaluations, Health / Mayo Clinic) frames the same shape at product scale. Both are retained here for the *eval-plane* platform; the *training-plane* platform stays with the training-pipeline track. | — |
| 18 | RAG engineering (chunking, embeddings, vector stores, rerankers) | n/a — peer track | `rag-engineer` (level 25) | mod-107 covers RAG *eval* methodology; RAG system engineering (servicenow-mee-01's primary skill) linked out | — |
| 19 | LLM application engineering (prompting, agents, tool design, product integration) | n/a — peer track | `llm-application-developer` (level 25) | Out of scope — mentioned only as the subject under evaluation | — |
| 20 | Alignment-risk methodology (harm modelling, red-team data generation) | n/a — peer track | `ai-risk-engineer` (level 30) | mod-109 owns measurement; harm-model design and red-team data generation linked out. apollo-mee-01/02's "alignment faking and scheming" is retained here for the detection/measurement side, deferred for the harm-modelling side. | — |
| 21 | Deep ML/AI security (model extraction, eval-set exfiltration, supply-chain attacks on judges) | n/a — higher level | `ai-infra-security-learning` (level 35) | Surfaced as awareness in mod-102 and mod-109; depth owned upstream. salesforce-mee-01 sits mostly here and is retained only for its eval-pipeline-construction requirement. | — |
| 22 | Governance / compliance / policy / dataset licensing review depth | n/a — peer track | `ai-governance-analyst` (level 25) | Surfaced as awareness in mod-102 / mod-112; depth owned upstream | — |

### Reading the evidence-frequency numbers

- **Denominator = 31** (postings sampled this cycle). The window closed on 2026-09-03; if a requirement is not cited by ≥30% (≥10/31) of postings, the packet does not authorise net-new curriculum content for it, even if the requirement is widely accepted as important. See [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json) for the empty-arrays proposal.
- **Low-frequency but essential-foundation requirements** (mod-103 classical eval at 0.10; mod-106 human eval at 0.13) are retained at full depth because they underlie higher-frequency work. Postings sample the surface (what the JD asks about); the curriculum teaches the foundation and the surface. Do not demote without cross-referencing peer-track posting evidence.
- **Highest-frequency requirements this cycle**: eval platform engineering (0.68 — up from 0.60), agent & tool eval (0.61 — up from 0.57), benchmark engineering (0.58), evaluation methodology (0.48 — up from 0.40), safety/red-team (0.39 — up from 0.33), production regression (0.35).
- **Denominator changed vs last cycle** (was 30, now 31). Direct year-over-year frequency comparisons should divide the deltas by roughly `(current/31 - previous/30)`, not treat 0.01–0.02 movements as material.

## Posting evidence — headline table

Full verbatim requirement bullets, preferred bullets, salary bands, source URLs, and per-posting notes live in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) under `postings[]`. Summary here for orientation.

| posting_id | Employer | Title | Date posted | Location | Salary (as published) |
|---|---|---|---|---|---|
| anthropic-mee-02 | Anthropic | Research Engineer, Model Evaluations | 2026-07-10 | SF / NYC / Remote-Friendly | $500K – $850K |
| openai-mee-01 | OpenAI | Research Engineer, Frontier Evals & Environments | est. 2026-07 | San Francisco, CA | $205K – $380K |
| openai-mee-02 | OpenAI | Research Engineer, Frontier Evals & Environments – Finance | est. 2026-06 | San Francisco, CA | $200K – $370K |
| apple-mee-01 | Apple | Software Engineer, Agentic Evaluation | 2026-06-23 | Cupertino, CA | $147K – $272K base |
| nvidia-mee-01 | NVIDIA | Evaluation and ML Systems Engineer, AI Safety and Security Engineering | 2026-07-27 | Santa Clara, CA (remote-eligible) | Not published |
| meta-mee-01 | Meta MSL | Research Engineer, Evaluations | est. 2026-07 | Menlo Park, CA | Not published |
| deepmind-mee-01 | Google DeepMind | Research Engineer, Frontier Safety Risk Assessment | est. 2026-07 | London, UK | $136K – $245K |
| scale-mee-01 | Scale AI | AI Research Engineer, Enterprise Evaluations | est. 2026-05 | SF / NYC / Seattle | $179K – $224K |
| nous-mee-01 | Nous Research | Machine Learning Engineer, Evals | 2026-07-14 | NYC (Remote) | Not published |
| ellipsis-mee-01 | Ellipsis Health | AI Evaluations Engineer – Healthcare | est. 2026-06 | Remote, US | $150K – $180K |
| mercor-mee-01 | Mercor | Research Engineer – Benchmarking, Evals & Failure Analysis | est. 2026-08 | SF (onsite) | $180K – $500K |
| triomics-mee-01 | Triomics | ML Model Evaluation Engineer | 2026-07-03 | US/India hybrid | Not published |
| surge-mee-01 | Surge AI | Software Engineer, Coding Evaluation & Training Data | est. 2026-06 | Remote-friendly | Not published |
| apollo-mee-01 | Apollo Research | Research Scientist/Engineer (Evaluations) | 2026-07-26 | London, UK or SF | £100K – £200K (London) |
| apollo-mee-02 | Apollo Research | Evals Software Engineer | est. 2026-07 | London, UK or SF | Not published |
| aisi-mee-01 | UK AISI | Software Engineer, Core Technology Team | est. 2026-08 | London (multi-site UK) | £65K – £145K + 28.97% pension |
| metr-mee-01 | METR | Member of Technical Staff, Evaluation Execution | est. 2026-06 | Berkeley, CA | $285K – $503K |
| epoch-mee-01 | Epoch AI | Software Engineer, Benchmarking | est. 2026-06 | Remote (global) | $125K – $275K |
| epoch-mee-02 | Epoch AI | Researcher, Evaluations | est. 2026-06 | Remote (PT/GMT overlap) | $115K – $200K |
| cais-mee-01 | Center for AI Safety | Research Engineer / Scientist | est. 2026-07 | San Francisco, CA | $140K – $200K |
| google-mee-01 | Google | Software Engineer, AI Evaluations (Pixel/Android) | est. 2026-07 | United States | Not published on posting |
| google-mee-02 | Google | Software Engineer, AI Quality and Benchmarks | est. 2026-07 | United States | Not published on posting |
| google-mee-03 | Google | Software Engineer, On-device AI Model Evaluation | est. 2026-07 | United States | Not published on posting |
| google-mee-04 | Google Research / Health AI | Research Software Engineer, Generative AI Evaluations, Health AI | est. 2026-07 | United States | Not published on posting |
| microsoft-mee-01 | Microsoft AI | Member of Technical Staff, AI Evaluations, Health (Mayo Clinic embed) | est. 2026-08 | New York, NY (onsite) | IC4 $119K–$261K; IC5 $142K–$304K (nat.–NYC bands) |
| servicenow-mee-01 | ServiceNow (Moveworks) | ML Engineer, GAI Search Platform | est. 2026-07 | Mountain View, CA | $139K – $216K |
| salesforce-mee-01 | Salesforce (Trust / AI Research) | Adversarial AI & Research Engineer | est. 2026-06 | Multi-site US (Remote / SF) | $148K – $246K (SF/NYC) |
| firecrawl-mee-01 | Firecrawl | Research Engineer (Evals) | 2026-07-12 | SF (Hybrid) | $210K – $275K |
| whitecircle-mee-01 | White Circle | Research Engineer (Evals) | 2026-06-28 | Paris (Hybrid) / London | $120K – $250K |
| innodata-mee-01 | Innodata Inc. | Applied Research Scientist, LLM Evaluation & Post-Training | est. 2026-06 | Remote, Canada | CAD $245K – $315K |
| glean-mee-01 | Glean | Software Engineer, Evals | est. 2026-07 | Bangalore, India | Not published on posting |

Notable postings considered but excluded per the sampling rules this cycle:

- `anthropic-mee-01` (prior cycle) — Anthropic RE Model Evaluations at $320–485K, posted 2026-05-05 — outside the 90-day window. The equivalent Anthropic role is retained via `anthropic-mee-02` at the newer 2026-07-10 date and $500–850K band.
- `agi-inc-mee-01` (prior cycle) — AGI Inc. RE Evals, posted 2026-05-27 — 9 days outside the window.
- `braintrust-mee-01` (prior cycle) — Braintrust Eval Engineer — canonical URL now 404s; no equivalent IC-level Braintrust eval-engineer opening currently listed. Removed rather than retained.
- Sentry "Senior SWE, AI Evals" ($155K–$400K) — Senior title excluded per rule (i). Notable as a first-time entry from an APM/observability incumbent (outside pure LLMOps vendors) into eval-engineer hiring.
- Artificial Analysis "Senior AI/ML Engineer (Benchmarking Stack)" (London/Sydney) — Senior title excluded per rule (i).
- P-1 AI "AI Evals Technical Lead" ($200K–$250K) — Lead title excluded per rule (i).
- Bloomberg "Senior SWE — AI App Enablement & Observability" (Dublin, 2026-04-28) — Senior title AND out of window.
- xAI "Member of Technical Staff — Model Evaluation" (London, 2026-03-18) — 79 days outside window.
- ByteDance "AI Product Manager (Evaluation), Flow" (Singapore, 2026-05-24) — PM (not engineering) title; also just outside window.
- Mercor "SWE AI Data & Evaluation" ($130K–$500K, 2026-05-29) — 7 days outside window; equivalent Mercor role retained via `mercor-mee-01` at newer date.

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
| `model-evaluation-engineer` (this, level 30) | ML Engineering | Benchmark engineering depth, statistical methodology, multi-modality (classical / LLM / multimodal / agent), eval platform architecture | The 31 postings in this packet |
| `ai-eval-engineer` (level 25) | AI Engineering | Build-altitude LLM/agent harness practitioner work inside product apps (trajectory eval, eval-gated CI, eval dashboards for app teams) | App-embedded eval postings |
| `ai-evaluation-engineer` (level 25) | Governance | Release-assurance / governance shape — audit trails, regulator-facing docs, third-party evaluator interface | Regulator-facing / QMS postings |

If a posting emphasises **benchmark construction, statistical rigour, eval platform internals, or cross-modality measurement at depth**, route it here. If it emphasises **product-side eval pipelines, prompt CI, or app-team enablement**, route it to `ai-eval-engineer`. If it emphasises **audit trails, regulator interface, or release-governance evidence**, route it to `ai-evaluation-engineer`.

## Emerging patterns below threshold

These patterns appear in the sample but fall below the 30% frequency threshold that would authorise net-new curriculum content this cycle. They are recorded here so the next cycle can pick them up if the frequency rises; each links to the adjacent existing coverage that would extend to absorb them if promoted.

1. **RL environment authoring for capability elicitation** — the eval doubles as an RL training environment; the eval engineer authors reward-generating environments (not just scoring rubrics) that steer the training run. **Freq: 6/31 (0.19)**, up from 3/30 (0.10). Postings: `openai-mee-01`, `openai-mee-02`, `meta-mee-01`, `surge-mee-01`, `mercor-mee-01`, `nous-mee-01`. Adjacent coverage: [`mod-108-agent-and-tool-eval`](lessons/mod-108-agent-and-tool-eval) already covers sandboxed trajectory scoring and partial-credit rubrics; environment authoring would be an exercise extension there, not a new module. Growth this cycle is real (frontier labs + Surge + Mercor) but still below threshold; watch next cycle.
2. **First-class eval dashboards and visualization** — dashboards researchers *and* leadership trust for training-run decisions. **Freq: 6/31 (0.19)**, down from 7/30 (0.23) after losing anthropic-mee-01 and agi-inc-mee-01 to window rules. Postings: `anthropic-mee-02`, `firecrawl-mee-01`, `glean-mee-01`, `google-mee-02`, `metr-mee-01`, `openai-mee-01`. Adjacent coverage: [`mod-111-eval-platform-engineering`](lessons/mod-111-eval-platform-engineering) already covers the data-warehouse-and-orchestration substrate; dashboarding would be a UI/UX-of-eval extension exercise.
3. **On-device / mobile / OEM-hardware ML eval** — quantized-model quality regression, latency-vs-quality on device, cross-hardware benchmark reproducibility. **Freq: 3/31 (0.10)**, down from 4/30 (0.13) with agi-inc-mee-01 out. Postings: `apple-mee-01`, `google-mee-01`, `google-mee-03`. Adjacent coverage: [`mod-110-production-eval-regression`](lessons/mod-110-production-eval-regression) already covers MLPerf-style serving benchmarks; an on-device exercise would extend that module.
4. **Agent trace-level observability** — span enrichment, durable telemetry, debugging workflows for agent behavior. **Freq: 5/31 (0.16)**, up from 4/30 (0.13). Postings: `glean-mee-01`, `ellipsis-mee-01`, `nous-mee-01`, `anthropic-mee-02`, `microsoft-mee-01`. Adjacent coverage: mod-110 already integrates Arize Phoenix / Langfuse / W&B Weave. Not a new-content signal this cycle.
5. **Domain-specialised eval (finance, healthcare, clinical)** — eval expertise *in* the vertical rather than generic LLM eval alone. **Freq: 5/31 (0.16)**, up from 4/30 (0.13). Postings: `openai-mee-02` (finance), `google-mee-04` (health), `ellipsis-mee-01` (health), `triomics-mee-01` (clinical), `microsoft-mee-01` (health, Mayo Clinic embed). Adjacent coverage: [`project-101-benchmark-engineering-capstone`](projects/project-101-benchmark-engineering-capstone) already lists a domain menu (medical Q&A, legal summarisation, code review, customer support, analyst data Q&A) — no change needed unless the frequency rises.
6. **"Failure analysis" as a first-class title concept** — treating diagnosis of *why* a model fails on an eval as a distinct engineering competency, not just a byproduct of running benchmarks. **NEW this cycle. Freq: 3/31 (0.10)**. Postings: `mercor-mee-01` (title literally includes "Failure Analysis"), `nous-mee-01` ("turn model failures into measurable signals"), `anthropic-mee-02` ("debug anomalous eval results mid-training-run"). Adjacent coverage: partial-credit / trajectory diagnosis in [`mod-108`](lessons/mod-108-agent-and-tool-eval) and regression triage in [`mod-110`](lessons/mod-110-production-eval-regression); if this frequency rises, add a cross-cutting exercise thread rather than a new module.

## What the next cycle should check

- Rerun the same 5-segment fan-out. Watch whether **RL environment authoring** (now 0.19) and **failure analysis as first-class** (now 0.10) grow toward the 0.30 threshold; if either crosses, add an exercise inside mod-108 (env authoring) or a cross-cutting exercise in mod-108 + mod-110 (failure analysis) — not new modules.
- Recheck **classical ML eval depth** (0.10 this cycle) against fine-tuning-engineer and senior-ml-engineer posting evidence before considering demotion — this module's low direct citation is expected in an LLM-dominated sample but may still be foundational.
- Widen geographic coverage. This cycle still skewed heavily US/UK/EU. Explicitly probe India (Glean, Triomics are the only signals), APAC (Singapore ByteDance, Sydney Artificial Analysis excluded on seniority), and LATAM enterprise-eval postings in the next fan-out.
- Recheck whether hyperscaler eval hiring (Amazon Bedrock, Databricks Mosaic, Bloomberg AI, JPMorgan AI Research) has any *non*-Senior/Staff/Principal shape appearing — they contributed zero postings this cycle because their published roles carry filtered-out modifiers.
- Confirm CAISI (US AI Safety Institute) status. All six teams show "not hiring at this time" on the careers page as of 2026-09; the UK AISI stays a strong signal. If CAISI reopens with an eval-engineer-titled opening, add to next cycle's safety-institute segment.
- Watch Microsoft AI's org-chart shift: the prior MAI Superintelligence Evaluations Engineer role (URL now 404) has been replaced by the Mayo Clinic health-vertical eval role. If Microsoft AI publishes a *distinct* superintelligence-team eval-engineer opening again next cycle, the domain-specialized-eval pattern separates into two Microsoft signals rather than one.
- Track whether "failure analysis" as a *title-level* concept spreads beyond Mercor. If a second employer uses it in the actual title (not just body text), the theme has clearly emerged as a job-market shape.
