# exercise-03: Dangerous-Capability Evaluation Walkthrough (Awareness Level)

**Estimated effort:** 6 hours

## Objective

Run at least three awareness-level dangerous-capability evaluations against a candidate model — one per public risk framework domain (cyber, autonomy, CBRN) — and produce a per-domain report that maps each score to a threshold under a public risk framework (Anthropic RSP v2.0, OpenAI Preparedness Framework, or UK AISI evaluation model). The deliverable is *not* a threshold-crossing determination (that decision is human governance work). It is a *reproducible eval + framework mapping* that a governance partner could consume to inform their decision. The eval is strictly awareness level — no dangerous outputs produced, no domain-specific attack authorship, no wet-lab reasoning.

## Prerequisites

- mod-109 Chapter 4 (dangerous-capability evaluations, risk frameworks).
- mod-104 Chapter on eval harnesses (`lm-eval-harness`), mod-108 Chapter 6 (Inspect agent harness) for the agent-shaped evals.
- Familiarity with at least one of the three risk-framework documents at the level of "which categories does it name and what are the qualitative thresholds."
- Python 3.11+; enough compute to run WMDP (small, cheap) and a small agent eval (Cybench task subset — moderate).

## The instruments

Pick one benchmark per domain. Minimum defensible set:

- **CBRN / hazardous knowledge (proxy).** WMDP (Li et al. 2024): three splits (bio, chem, cyber-security). Load from `cais/wmdp` on Hugging Face, pin the revision SHA. Use `lm-eval-harness` (mod-104) with the `wmdp` task; run at 5-shot per the paper's protocol.
- **Cyber offense (agentic).** Cybench (Zhang et al. 2024) *or* an InterCode-CTF subset *or* CyberSecEval 3 (Meta, 2024). Run a 30–60 task subset via an Inspect agent harness (mod-108 Chapter 6). Report task-solve rate.
- **Autonomy / long-horizon.** METR public task suite (`metr-evals/tasks-public` or similar; the exact repo name is subject to move — verify at run time) OR use SWE-bench Verified from mod-108 exercise-02 as a lower-bound autonomy proxy. Report solve rate.

If any of these benchmarks moves URL or gets a substantially updated version between now and your run, use the current maintained version and note the substitution in the report. The *domain coverage* is the acceptance criterion, not any single benchmark's name.

## Requirements

### Part A — run the evaluations

Build `capability_eval/` with:

- **`capability_eval/wmdp.py`** — a thin wrapper around `lm-eval-harness` for the three WMDP splits. Pin the dataset revision, log the model version, temperature, and shots.
- **`capability_eval/cyber_agent.py`** — an Inspect-based agent runner for a Cybench (or equivalent) subset. Follow the mod-108 Chapter 3 discipline: log the agent harness version, the budget cap, the sandbox configuration, the tool descriptions.
- **`capability_eval/autonomy.py`** — either a METR task-suite runner (if available) or a SWE-bench-Verified subset runner reused from mod-108 exercise-02. In either case, log the agent harness config.
- **`capability_eval/aggregate.py`** — per-domain aggregation with 95% bootstrap CIs, per-category slicing where the benchmark supports it (WMDP-bio breakdown by topic, Cybench breakdown by difficulty tier, autonomy breakdown by task length).

### Part B — the framework mapping

Pick *one* public risk framework as your primary target. Options:

- **Anthropic RSP v2.0** (published October 2024, subject to further updates — check for a current version at the time of your run).
- **OpenAI Preparedness Framework v1.0 or later.**
- **UK AISI evaluation protocol** (as documented in AISI's public evaluation summaries).

For each domain (cyber, autonomy, CBRN), produce a *mapping paragraph* that:

1. States the score with CI.
2. Cites the specific framework version and the specific threshold-language passage the score is mapped against.
3. Records the *qualitative* mapping: "this score is well below / near / at the threshold for [named threshold]."
4. Records the *confidence* in the mapping ("HIGH: score is 40 points below a well-documented threshold; benchmark is a well-validated proxy" versus "LOW: score is close to a poorly-specified threshold; benchmark is a proxy of unknown fidelity") with a one-sentence justification.

The mapping is qualitative. Do not invent numeric cutoffs the framework does not publish. If the framework does not give a numeric cutoff, the mapping paragraph says so.

You are producing an *input* to a governance decision, not making the decision. The final sentence of every mapping paragraph reads (some variant of): "Governance review by [named body] is required for the release decision. This report is an evaluation-side input only."

### Part C — the report

```
# Dangerous-capability evaluation (awareness level)

Model:               <provider/model@version>
Framework:           Anthropic RSP v2.0 (2024-10)
                     [reference: full document available under NDA / linked from official page]
Evaluation date:     <YYYY-MM-DD>
Governance review:   Deferred to Safety Board meeting <date>. This report informs; it does not decide.

## Cyber offense
Instrument:   Cybench (rev <sha>, 60 tasks, difficulty tiers T1-T4)
Harness:      Inspect + basic_agent, budget 40 steps, sandbox <docker-digest>
Metric:       Task solve rate 0.32 [0.27, 0.37]
              T1: 0.62, T2: 0.38, T3: 0.14, T4: 0.02

Framework mapping (RSP v2.0):
  Threshold: "meaningfully assists non-experts in offensive cyber operations"
             (qualitative; RSP §<section>).
  Assessment: Well below threshold. T3/T4 scores near zero suggest no
              non-expert-relevant capability on non-trivial tasks.
  Confidence: HIGH. Benchmark validated against expert-solve baseline;
              per-tier breakdown is coherent.

## Autonomy
Instrument:   METR task suite (public split, rev <sha>, N tasks)
              + cross-reference: SWE-bench Verified score 0.51 from capability report
Harness:      Inspect + basic_agent, budget 100 steps
Metric:       METR solve rate 0.24 [0.19, 0.29]
              By horizon (steps to solve for a human reference): <breakdown>

Framework mapping (RSP v2.0):
  Threshold: "autonomous replication and adaptation" (qualitative; RSP §<section>).
  Assessment: Well below threshold. Model succeeds on short-horizon tasks
              but degrades rapidly beyond ~15-step trajectories.
  Confidence: MEDIUM. METR public suite has known coverage gaps; a wider
              in-scope evaluation is recommended before threshold review.

## CBRN / Biosecurity (proxy)
Instrument:   WMDP-bio 5-shot, 0.62 [0.59, 0.66]
              WMDP-chem 5-shot, 0.58 [0.55, 0.61]
              WMDP-cyber 5-shot, 0.54 [0.51, 0.57]
Cross-reference:
  Internal biosecurity uplift study (Study <ID>, n=40): scheduled 2026-Q3;
  results not available for this report.

Framework mapping (RSP v2.0):
  Threshold: "meaningfully increases risk of a bioweapons attack by non-experts"
             (qualitative; RSP §<section>). Cited in evaluation as requiring
             both knowledge-recall proxy AND direct uplift evidence.
  Assessment: WMDP-bio proxy indicates non-trivial hazardous-knowledge recall.
              Threshold determination REQUIRES the uplift study; this report
              alone is not sufficient to inform the safety-board decision.
  Confidence: LOW. Proxy-only; uplift evidence not yet available.

## Limitations
- Only three domains covered; persuasion not evaluated in this run.
- Benchmark set is public and subject to contamination in pretraining;
  a private held-out set is a stretch goal.
- Framework mapping is qualitative and depends on human review.
- Uplift studies (the direct evidence for CBRN threshold decisions) are
  scheduled but not available for this report.

## Reproducibility manifest
Model: <provider/model-id> at <version-string>
Harness: Inspect <version>, lm-eval-harness <version>
Datasets:
  - WMDP: revision <sha>
  - Cybench: revision <sha>
  - METR public suite: revision <sha>
Sandbox: docker <digest>
Run identifier: <uuid>
Machine: <cpu>, <ram>, <region>
```

### Part D — data handling

Ship `docs/DATA_HANDLING.md` describing:

- Where the eval logs live and who has access.
- Retention: the *aggregate* metric report is retained; the *per-item response* data (particularly for WMDP-bio) is treated as sensitive and is deleted after 30 days unless flagged for follow-up.
- A "no dangerous completions in report" rule: if a WMDP-bio item's response happens to include hazardous instruction-following text, that response goes to the controlled store, never to the report or the eval log.
- Access-audit for the controlled store.

## Starter guidance

- **Read the framework document first.** Whichever framework you pick, spend an hour reading the actual public text before you start writing mapping paragraphs. The mapping is only as good as your understanding of what the framework actually says.
- **Do not invent numeric thresholds.** RSP v2.0 does not publish "WMDP-bio > 70% = ASL-3." If the framework language is qualitative, your mapping is qualitative — do not write false precision into the report.
- **Cite the section, not the general document.** "RSP v2.0 §X.Y" is a citation; "RSP" is a wave in the direction of a document. If the framework does not have numbered sections, cite the page or heading.
- **The uplift study matters for CBRN.** WMDP alone is a proxy. If your organization has run or plans to run an uplift study, cite it (schedule + report locus). If it has not, your CBRN mapping's confidence field must reflect the proxy-only status.
- **This is a governance artifact, not an engineering artifact.** Draft it in a shape that a Safety Board member could read. Numbers with CIs, mapping paragraphs, confidence assessments, limitations, and reproducibility manifest.
- **You are not the decider.** Every domain section ends with a reminder that governance review is required. If you find yourself writing "this model is safe" or "this model exceeds the threshold" in your voice, back up — that language belongs to the governance body, not the eval report.
- **Version everything.** Framework version, benchmark revision, harness version, sandbox digest, model version. mod-102 discipline; mod-108 discipline; combined at higher stakes.

## Acceptance criteria

- At least three domains covered (cyber, autonomy, CBRN), each with at least one publicly-cited benchmark and a run report.
- Every benchmark score has a 95% bootstrap CI (mod-101 discipline).
- Framework mapping paragraph exists for each domain, citing a specific framework version and a specific threshold-language passage (section / page). Confidence field is HIGH / MEDIUM / LOW with a one-sentence justification.
- Report contains a limitations section that explicitly names uncovered domains and instrument gaps.
- Reproducibility manifest pins model version, harness version, all dataset revisions, and sandbox configuration.
- `docs/DATA_HANDLING.md` describes retention and access-audit rules for eval logs and, separately, for any dangerous-completion artifacts.
- The report never states a threshold-crossing decision in the drafter's voice; every decision is deferred to a named governance body.

## Stretch goals

- **Framework A/B.** Produce the mapping under a second framework (e.g., you mapped under RSP; also map under Preparedness Framework). Compare where the two frameworks disagree on category boundaries. This is a durable governance-literacy exercise.
- **Uplift study lite.** With policy sign-off, run a small (n=15) mock uplift study in one CBRN domain: humans-with-web-search versus humans-with-model on a set of *benign* scenarios (do not run the study on genuinely dangerous scenarios without IRB-equivalent review). Report the delta and comment on what a real uplift study would need to inform the CBRN threshold.
- **Cross-model comparison.** Run the same three benchmarks against two models (e.g., a small OSS model and a hosted frontier model). Report the per-domain gaps. Frontier models will be higher on CBRN proxies; the cross-comparison is what turns "62% on WMDP-bio" from an isolated number into an interpretable one.
- **Private held-out.** Author, under CBRN policy team review, a small (n=50) private held-out biosecurity multiple-choice set that does not overlap WMDP. Score both and report the gap. A larger gap on the held-out set is a contamination signal.
- **External-evaluator dry run.** Draft the summary that would go to an external evaluator (UK AISI or an enterprise customer's compliance team) using only content from your run, and have a colleague review it under the "would this satisfy an auditor?" test.
