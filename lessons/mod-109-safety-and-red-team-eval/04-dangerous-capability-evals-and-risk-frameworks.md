# Dangerous-Capability Evaluations at the Awareness Level

Refusal and jailbreak resistance measure the model's *behavior* under a request. Dangerous-capability evaluation measures something else: given that the model *does* comply — because it was jailbroken, or because a future release removed some mitigation, or because the developer bypassed the safety layer — how much *uplift* does the compliance actually provide to someone trying to cause harm? A refusal-rate number by itself does not answer this: a model that refuses 99% of the time but delivers actionable synthesis instructions in the 1% that leaks is not equivalent to a model that "refuses" 99% of the time by producing gibberish in the 1% that leaks. The severity axis is what distinguishes the two.

This chapter walks the *awareness-level* form of dangerous-capability eval that a governance / risk partner uses to make a release-time judgement, and the three public risk frameworks — Anthropic's Responsible Scaling Policy (RSP), OpenAI's Preparedness Framework, and the UK AI Safety Institute (AISI) evaluation program — that turn a benchmark score into a decision input. It is deliberately *not* a walkthrough of how to actually cause the harms the eval measures; the exercises and the citations stay at the level where a reader with mod-104 skills can *run* a public benchmark, *aggregate* the result, and *map* it into a framework, without ever authoring domain-specific attack content.

## What "dangerous capability" means in this context

The public frameworks converge on three or four risk domains, with slight naming differences:

- **Cyber offense.** Can the model meaningfully assist a moderately-skilled attacker in exploit development, offensive tool use, or malware authorship? Reference public evals: Cybench (Hendrycks-adjacent, 2024), NYU CTF (Hackathon-derived), the InterCode-CTF suite, and the CyberSecEval series (Meta, 2023 onward).
- **Autonomy / self-directed action.** Can the model plan and execute multi-step tasks (including agent-shaped tasks with tools) at a level that suggests it could act independently on a hostile objective? Reference public evals: METR's task suite (Model Evaluation and Threat Research, formerly ARC Evals — a battery of long-horizon software-engineering and reasoning tasks), the RE-Bench autonomy suite, and SWE-bench Verified (mod-108) used as a lower bound on realistic-workload agent capability.
- **Biosecurity, chemistry, radiological / nuclear (CBRN).** Can the model provide meaningful uplift on hazardous knowledge that is otherwise gated by expert training or restricted-access literature? Reference public evals: WMDP (Weapons of Mass Destruction Proxy, Li et al. 2024), which uses multiple-choice questions in bio, chem, and cyber as *proxies* for the underlying dangerous knowledge, deliberately built so that the questions themselves are not dangerous but performance on them correlates with dangerous knowledge.
- **Persuasion and manipulation** (sometimes a separate domain, sometimes folded into misuse). Can the model produce meaningfully more persuasive content than a strong human baseline, across long conversations, in ways that could affect elections, radicalize, or facilitate scams? Reference: various academic persuasion studies, some anchored to public model cards' persuasion sections.

Each domain has its own instruments, its own way of being misleading, and its own mapping into a risk framework. This module treats the first three (cyber, autonomy, CBRN) as the core awareness-level set; persuasion is a legitimate fourth but is less standardized at time of writing.

## The awareness-level constraint

An awareness-level eval satisfies three properties:

- **The eval itself is not dangerous.** WMDP is the canonical example: the multiple-choice questions and their answers are drawn from open literature, and the *test* is whether the model can pattern-match the hazardous knowledge, not whether the model produces a novel hazardous artifact. Running WMDP does not create hazardous outputs.
- **Performance on the eval correlates with, but does not equal, real hazard capability.** A high WMDP-bio score is *evidence* that the model has absorbed enough biosecurity literature that a determined uplift-seeking user might succeed with it. It is not *itself* the ability to synthesize a pathogen; that boundary requires domain-expert judgement, wet-lab access, and many other steps the model cannot perform.
- **The eval is intended as an input to human review, not an autonomous decision.** The threshold that turns "WMDP-bio = 65%" into "trigger the ASL-3 deployment standard" is set by policy, and the crossing is confirmed by a governance body — the safety board, the release-review committee, or the external evaluator (like UK AISI). The eval is a tripwire; the decision is human.

The *distinction* to internalize: a capability eval and a dangerous-capability eval are the same shape (dataset + scorer + report) but they answer different questions ("how well does the model do X" versus "how close is the model to a risk threshold on X"), and they are consumed by different processes. Chapter 1's dashboard placed dangerous-capability eval as its own line item precisely because collapsing it into a capability-eval slot loses the risk-framework connection.

## The three public risk frameworks and their thresholds

At time of writing, three publicly-documented risk frameworks are load-bearing for how frontier labs and evaluators structure release decisions.

### Anthropic's Responsible Scaling Policy (RSP)

The RSP (Anthropic 2023, v1.0; substantially updated in v2.0 in October 2024) defines a series of "AI Safety Levels" (ASL-2, ASL-3, ASL-4, ...) — analogous to biosafety levels — each associated with a set of "capability thresholds" and a corresponding "deployment standard" and "security standard." Concretely:

- **ASL-2** covers current-generation LLMs; the deployment standard is the usual industry safety and content practices.
- **ASL-3** is triggered by specific *capability thresholds* (e.g. "meaningfully increases the risk of a bioweapons attack by non-experts" or "can autonomously replicate and acquire resources") and requires stronger deployment and security standards (RSP v2.0 spells out specific ones).
- **ASL-4 / ASL-5** are placeholders for future capability levels; the thresholds are described but the exact operating rules are not yet fixed.

For an eval engineer, the relevant question is: *which of my benchmark scores, if it crosses a specific value, triggers a threshold?* Anthropic's public documentation names the evaluation domains it uses to test the thresholds (biosecurity uplift studies, cyber offense capabilities, autonomous replication and adaptation, model self-exfiltration) but publishes the *thresholds* qualitatively (in the form of "if a model can do X, then Y") rather than as specific benchmark cutoffs. The evaluator's job is to run the relevant benchmarks and *inform* the threshold judgement; the threshold judgement itself is made by Anthropic's Responsible Scaling Officer and safety board.

### OpenAI's Preparedness Framework

The Preparedness Framework (OpenAI, December 2023, v1.0; updated 2024–2025) defines *four risk categories*: cybersecurity, biological / chemical / radiological / nuclear (CBRN), persuasion, and model autonomy. Each category has a *risk level* — Low, Medium, High, Critical — with explicit descriptions of the capabilities that put a model at each level. The framework also names:

- A set of *pre-deployment evaluations* (some public, some internal) that produce a score per category.
- A *scorecard* format that reports the four category scores in a shared shape.
- A rule that *no model with a post-mitigation score of "High" or above in any category ships* without explicit safety-committee approval, and *no "Critical" model ships* absent unprecedented mitigations.

The Preparedness Framework is the most explicit public commitment to "score-to-decision" wiring: the mapping from eval outputs to release action is written down. For eval engineers this is instructive — it is a reference point for how a framework maps benchmark work into governance action, even for teams not directly working under it.

### UK AI Safety Institute (AISI)

UK AISI (established 2023, produced a first evaluation of pre-release models in 2024) is a *government* evaluator rather than a lab-internal one. Its evaluation model:

- Independent access to models pre-release, under contractual arrangements with the developers.
- Focused evaluation of four to five capability areas broadly matching the RSP / Preparedness set (cyber, biosecurity, autonomy, agentic tasks, plus safeguards / robustness).
- Public writeup of methodology and (usually aggregate) results, with more detail retained in confidential reports to the developer and to government partners.

The differences from lab-internal evaluators are structural (independence, government backing, ability to publish independently) not methodological (they use HarmBench-family tools, agent benchmarks, CBRN proxies). For an eval engineer at a developer lab, AISI's evaluation is *another instance* of the framework question: they will apply their own thresholds, and your job is to run comparable internal evaluations and interpret gaps.

There are analogous emerging bodies in the US (the US AI Safety Institute Consortium under NIST), Japan, Singapore, and the EU under the AI Act. The specific institutional map is moving; the *pattern* — independent evaluator, standardized categories, publishable methodology — is durable.

## What the eval engineer actually runs

Concretely, at the awareness level, a defensible mod-109 eval battery includes:

- **WMDP-bio, WMDP-chem, WMDP-cyber.** Multiple-choice question sets from Li et al. 2024. Loaded from Hugging Face, scored with a standard MCQ harness (mod-104 Chapter 2). The metric is accuracy per category, with mod-101 CIs.
- **A cyber-offense agentic eval.** Cybench (Zhang et al. 2024) is the current public reference: CTF-style tasks in a sandboxed environment, scored on "task solved / not solved," with an agent harness (mod-108). CyberSecEval 3 (Meta, 2024) is a comparable Meta-published suite. Report task-solve rate with an explicit agent harness spec (mod-108 Chapter 3 discipline).
- **An autonomy eval.** METR's task suite (public since 2023, expanded through 2024) or the RE-Bench autonomy set. Long-horizon software-engineering tasks graded as pass/fail with strong pass criteria. Report solve rate and per-task-length breakdown.
- **A model-organism / self-replication check.** Not a public benchmark exactly, but a shape: can the model, given a shell, some credentials, and a goal, do things like clone its own weights, spin up new processes, deploy code to an S3 bucket? These are usually run internal-only because the sandbox setup is delicate. If you cannot run them yourself, at least reference the METR literature that describes them.
- **Optional: persuasion.** Less standardized. If you include it, cite the specific published protocol and note that the field is early.

For each of these, the eval engineer's job is:

1. Run the benchmark. Pin the version, the harness, the model, and the budget (mod-104 / mod-108 discipline).
2. Compute the top-line metric with a CI.
3. Map the metric into the risk framework(s) your organization uses. The mapping is qualitative — the framework does not usually publish a specific "if benchmark X exceeds Y, escalate" line, and you are producing an input to a human judgement, not making the judgement yourself.
4. Report per-category (per-domain) not just per-benchmark. A single "dangerous capability" scalar is exactly the kind of aggregate that Chapter 1 warned against.

## The mapping problem: benchmark to threshold

The interesting difficulty in a dangerous-capability report is *not* running the benchmark; it is defending the mapping from a benchmark score to a threshold statement.

Two failure modes to be explicit about:

- **Over-claiming from a proxy.** WMDP is a proxy for hazardous knowledge, not a direct test of it. A high WMDP-bio score suggests the model has absorbed a lot of relevant literature, but the *actionable* claim "this model can uplift a novice to make a bioweapon" requires additional evidence (an uplift study — humans + model versus humans + web-search baseline, on structured tasks). Public uplift studies (RAND-adjacent work, Mouton et al. 2024 on biosecurity uplift, Anthropic's biosecurity uplift study cited in the RSP) are the actually-load-bearing evidence for a threshold decision. Your report should note that the benchmark is a proxy and cite the uplift study framework.
- **Under-claiming from a benchmark ceiling.** A benchmark score at ceiling (say, WMDP-bio at 90%) does not mean "the model has all the dangerous knowledge and any further improvement is impossible" — it means the *benchmark* saturates. When a benchmark ceiling is close, your report should stop citing it as the primary metric and pivot to a benchmark with more headroom or to an uplift-style study. Otherwise you are reporting a saturated instrument as "no further capability increase," which will be wrong.

The healthy report shape:

```
Domain: Biosecurity uplift
  Instruments used:
    - WMDP-bio (proxy): 62.4% [59.1, 65.7]  (dataset rev abc123, harness lm-eval v0.4.5)
    - Internal biosecurity uplift study, n=40 human participants:
      no meaningful uplift observed at p<0.05 vs web-search baseline (report link, controlled)
  Framework mapping:
    Anthropic RSP: below the "meaningfully increases risk of bioweapons attack by non-experts"
                   threshold (per internal review, dated 2026-07-15).
    OpenAI Preparedness (as reference framework): would map to "Low" per public definition.
  Confidence: MEDIUM. Uplift study n is small; expert review flagged three questions
              that WMDP-bio does not adequately cover.
```

This is *far* better than "biosecurity score: 62.4%". It puts the benchmark result in context, cites the framework it maps into, and calls out the confidence level in the mapping.

## Data-handling constraints that intensify at this level

The Chapter 3 data-handling rules apply here with extra weight.

- **Wet-content is off-limits internally as well.** A dangerous-capability report may cite the eval but must not include model outputs that constitute the dangerous information itself. If your eval logs contain the model's completions on the "how do you synthesize [X]" prompt, those logs are hazardous artifacts. They are stored in a much tighter access ring than jailbreak eval logs, with retention limits that are shorter, not longer.
- **External sharing is by summary only.** Model-card content for this section is aggregate scores, benchmark-version pins, and framework mappings, with methodology described at the level a governance partner needs. Detailed per-item verdicts stay internal. Chapter 7 has the shape.
- **Access to eval infrastructure is not a general-engineering credential.** Running a WMDP-cyber against a released model is fine (the dataset is public). Running an internal uplift study is *not* a general engineering activity; the participants and the setup usually need IRB-adjacent review, and the results are treated as sensitive.

If your organization does not yet have this data-handling wrapper around dangerous-capability eval work, Chapter 7's model-card discussion is a useful entry point for building one.

## Reporting a dangerous-capability section: minimum viable shape

For each domain (cyber, autonomy, CBRN), a defensible section includes:

- **Instruments used and versions.** Benchmark name, dataset revision, harness version, budget for agentic evals.
- **Top-line metric with CI.** From the mod-101 discipline. Slice by category if the benchmark supports it.
- **Comparison against relevant baselines.** Previous model version, other frontier models where publicly reported (with source date), or a task-specific human baseline where available.
- **Framework mapping.** Which threshold, in which framework, this score maps to. Cite the framework version.
- **Known limitations of the instrument.** Proxy versus direct test, ceiling effects, known gaps. Cite the benchmark author's own limitations statement where available.
- **Governance decision recorded elsewhere.** The eval report does not *make* the decision; it *informs* it. The report should say where the decision record lives (safety board minutes, release-review artifact, RSO signoff).

## Guidance for the eval author

- **Stay at the awareness level in what you author and what you publish.** Public benchmarks, standard scorers, aggregate metrics. No novel attack authorship; no reproducible harmful outputs.
- **Report by domain, not one composite.** Cyber, autonomy, CBRN, each with its own line and its own framework mapping.
- **Cite the framework version.** RSP v1.0 and v2.0 have different thresholds; Preparedness Framework v1 and v2 have restructured categories. A framework mapping is only interpretable next to the framework version.
- **Do not equate benchmark score with capability threshold crossed.** The mapping is qualitative and requires human review. Say so explicitly.
- **Cite uplift studies where they exist.** WMDP is a proxy; RAND/Anthropic-style uplift studies are the real evidence for CBRN thresholds. Where the studies exist, cite them; where they do not, name it as a limitation.
- **Coordinate the release path with governance partners.** The safety board, the RSO (or equivalent), and external evaluators (UK AISI, US AISI Consortium) all consume this report. Draft it with them, not at them.
- **Version everything.** Framework version, benchmark revision, harness version, model version. This is the mod-102 discipline at higher stakes.

## Summary

Dangerous-capability evaluation measures uplift severity — how much a compliant model would help someone cause harm — across a small number of domains (cyber offense, autonomy, CBRN) that public risk frameworks (Anthropic RSP, OpenAI Preparedness, UK AISI) have converged on. The eval discipline runs *at the awareness level*: public benchmarks (WMDP, Cybench, METR task suite) whose questions are not themselves dangerous, whose scores correlate with hazard capability, and whose interpretation is a human governance decision informed by (not made by) the eval. The reporting shape is per-domain, with a top-line metric, CI, framework mapping, and explicit limitations, plus a pointer to the governance decision recorded elsewhere. Data handling tightens further at this level — dangerous completions are hazardous artifacts, not eval logs; external sharing is aggregate-only. The next chapter turns to a different family of measurements — bias, toxicity, fairness — where the false-positive discipline and rater-bias discipline from mod-101 and mod-105 become the load-bearing skills.
