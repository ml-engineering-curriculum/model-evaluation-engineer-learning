# ML Engineering · Model Evaluation Engineer — Learning Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum). Live cohorts &amp; team programs: **[ai-infra-curriculum.github.io](https://ai-infra-curriculum.github.io/)**.
<!-- /aicg:site-banner -->

<!-- aicg:sponsor -->
> 💜 **[Sponsor this curriculum](https://github.com/sponsors/ml-engineering-curriculum)** — sponsorships keep the whole open-source AI Career Curriculum free and moving.
<!-- /aicg:sponsor -->

Measure models rigorously: eval-harness design, benchmark construction, statistical methodology, and regression detection across modalities.

> **Status**: curriculum plan authored (2026-06-25). Lessons and project READMEs will be drafted by subsequent autonomous content cycles. See [`CURRICULUM.md`](CURRICULUM.md) for the planned scope and [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) for the requirements-to-coverage map (postings list deferred to the next research cycle — see `JOB_REQUIREMENTS.md` Status section).

**Level:** 30 (deep specialist — peer to Senior ML Engineer on the ladder).
**Planned scope:** 12 modules (175h) + 3 projects (135h) = ~310 hours.

## What this track owns

Evaluation engineering end-to-end at depth, across modalities:

- Validity, sampling, and statistical methodology (point estimates with CIs, paired tests, FDR control)
- Benchmark engineering — sourcing, labelling with IAA, decontamination, versioning, canary sets, deprecation
- Classical ML eval depth — calibration, per-slice reporting, fairness measurement, operating-point selection
- LLM benchmark harnesses — `lm-evaluation-harness`, HELM, OpenAI Evals, Inspect (UK AISI), prompt-format sensitivity
- LLM-as-judge platforms — rubric design, bias controls (position / length / self-preference), Arena-style ELO
- Human evaluation — annotator workflows, agreement, gold-set rotation, vendor build-vs-buy
- Generative and multimodal eval — pass@k code, math, RAG (RAGAS / TruLens), multimodal
- Agent and tool-use eval — trajectory scoring, SWE-bench, WebArena, GAIA, AgentBench, METR, Inspect agent harness
- Safety and red-team eval — refusal / over-refusal / jailbreak / HarmBench, dangerous-capability eval methodology, prompt-injection robustness
- Production eval and regression detection — offline gates, shadow, A/B with CUPED, sequential testing, drift, MLPerf
- Eval platform engineering — versioned registry, multi-runner orchestration, eval data warehouse, CI integration, SLO
- Eval systems design — release-gate plans, model cards, NIST AI RMF / ISO 25059 / EU AI Act mapping, build-vs-buy

## What this track defers

- Build-altitude ML / PyTorch / classical-eval fundamentals → [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (level 20).
- Build-altitude LLM/agent eval-harness practitioner work in product pipelines → [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (level 25, AI Engineering family).
- Release-assurance / governance shape (audit trails, regulator interface) → [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (level 25, Governance family).
- Post-training depth (SFT / PEFT / RLHF / DPO) → [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning).
- Distributed-training PLATFORM engineering → [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning).
- Alignment-risk methodology and red-team data generation → [`ai-risk-engineer-learning`](https://github.com/ml-engineering-curriculum/ai-risk-engineer-learning).
- LLM application / RAG / NLP engineering depth → peer specialist tracks linked from `CURRICULUM.md`.
- Deep ML/AI security and governance / policy depth → [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning), [`ai-governance-analyst-learning`](https://github.com/ml-engineering-curriculum/ai-governance-analyst-learning), [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning).

## Layout

```
model-evaluation-engineer-learning/
├── .aicg/                    curriculum-plan.json and job-requirements.json (machine-readable catalog)
├── lessons/mod-XXX-*/        modules with lectures, exercises, labs, quizzes (to be drafted)
├── projects/project-XXX-*/   multi-module capstones (to be drafted)
├── CURRICULUM.md             role-level coverage map
├── JOB_REQUIREMENTS.md       requirements catalog with citations and ownership map
├── PREREQUISITES.md          assumed entry skills
├── VERSIONS.md               release history
└── README.md                 this file
```

## Paired Solutions Repo

[`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) carries the reference implementations.

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
