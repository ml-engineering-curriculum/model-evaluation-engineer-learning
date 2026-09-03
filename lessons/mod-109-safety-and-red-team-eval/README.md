# mod-109-safety-and-red-team-eval: Safety and Red-Team Evaluation

**Estimated effort:** 15 hours

Every prior module in the track asked variations of "did the model produce the right answer?" Safety and red-team evaluation asks a different family of questions — *did the model refuse when it should have refused; did it refuse when it should not have refused; how much attacker effort does it take to overturn a refusal; when it does comply, how much uplift did that give a hostile actor; and how close is the model to a threshold that requires a governance-level mitigation?* — and a capability-oriented scorer will silently answer these wrong. This module builds a dashboard of five loosely-orthogonal measurements (refusal on harmful, over-refusal on benign, jailbreak ASR against a documented attacker, dangerous-capability proximity to public-framework thresholds, prompt-injection robustness) plus the model-card writeup that ships those numbers to governance partners without leaking payloads.

The module operates strictly at the *awareness* level: you learn to run, score, interpret, and report against published attack suites and hazardous-knowledge proxies. You do not learn attack authorship, and no exercise produces raw payloads or harmful completions in an artifact that leaves the eval environment.

## Learning objectives

- Measure refusal rate, over-refusal rate, and jailbreak resistance against curated attack suites (HarmBench, XSTest, manual sets) without enabling attack content.
- Run dangerous-capability evals (cyber / autonomy / bio) at the awareness level, mapped to public risk frameworks (Anthropic RSP, OpenAI Preparedness Framework, UK AISI evaluation model).
- Measure bias / toxicity / fairness at the model level with false-positive-rate discipline and rater-bias sensitivity.
- Measure prompt-injection robustness with structured attack suites and explain the boundary with application-security testing.
- Author the safety-eval section of a model card that satisfies governance / risk partners without leaking attack payloads.

## Lecture chapters

1. [`01-why-safety-eval-is-its-own-discipline.md`](01-why-safety-eval-is-its-own-discipline.md) — the five-measurement dashboard, the two hard boundaries (awareness-level, external-artifact contents), and the reason "just measure accuracy" fails here.
2. [`02-refusal-and-over-refusal.md`](02-refusal-and-over-refusal.md) — the harmful and benign-lookalike sets, the compliance judge, the paired confidence interval, the two most common ways the measurement breaks (contamination, easy benign set), and category slicing.
3. [`03-jailbreak-resistance-with-harmbench.md`](03-jailbreak-resistance-with-harmbench.md) — attacker-model taxonomy (static, single-turn transformations, PAIR / TAP / Crescendo), the HarmBench instrument, the ASR-over-budget curve, the load-don't-author data-handling discipline, and the limits of ASR as a claim.
4. [`04-dangerous-capability-evals-and-risk-frameworks.md`](04-dangerous-capability-evals-and-risk-frameworks.md) — cyber / autonomy / CBRN domain coverage, WMDP / Cybench / METR as the awareness-level instruments, the mapping into Anthropic RSP, OpenAI Preparedness, and UK AISI evaluation, and the eval-versus-decision boundary.
5. [`05-bias-toxicity-and-fairness.md`](05-bias-toxicity-and-fairness.md) — BBQ / RealToxicityPrompts / BOLD as the anchor benchmarks, the classifier-calibration and human-gold-agreement discipline, the FPR patterns on AAVE / disability / LGBTQ+ language, rater bias in labelling data, and the mod-103 fairness discipline transferred to LLM outputs.
6. [`06-prompt-injection-robustness.md`](06-prompt-injection-robustness.md) — direct versus indirect injection, BIPIA / InjecAgent / Tensor Trust / AgentDojo suites, the *attack success at fixed benign utility* metric, tool-poisoning trajectory measurement, and the responsibility line with application-security testing.
7. [`07-authoring-the-safety-section-of-a-model-card.md`](07-authoring-the-safety-section-of-a-model-card.md) — the six-section public artifact skeleton (policy, refusal, jailbreak, dangerous-capability, bias/toxicity, injection, limitations), the exclusion list (raw payloads, per-item verdicts, novel exploits), the internal appendix under access control, and the governance-partner question test.

## Exercises

Five hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after the chapters it depends on.

- [`exercise-01-refusal-and-over-refusal-measurement.md`](exercises/exercise-01-refusal-and-over-refusal-measurement.md) — build the paired refusal / over-refusal pipeline against HarmBench standard + XSTest, with a calibrated judge, mod-101 CIs, per-category BH-adjusted slicing, and a paired-comparison delta against a baseline.
- [`exercise-02-jailbreak-resistance-with-harmbench.md`](exercises/exercise-02-jailbreak-resistance-with-harmbench.md) — run HarmBench against a candidate model with at least three attacker profiles (static, single-turn transformations, PAIR or TAP), publish an ASR curve over attacker budget, and ship the two data-handling policies (`DATA_HANDLING.md`, `EXTERNAL_ARTIFACT.md`) that keep the run from becoming an attack cookbook.
- [`exercise-03-dangerous-capability-eval-awareness-walkthrough.md`](exercises/exercise-03-dangerous-capability-eval-awareness-walkthrough.md) — run three awareness-level evals (WMDP for CBRN, Cybench for cyber, METR / SWE-bench Verified for autonomy) and produce per-domain framework-mapping paragraphs against one of Anthropic RSP v2.0, OpenAI Preparedness, or UK AISI evaluation model, with explicit confidence and governance-decision deferral.
- [`exercise-04-bias-and-toxicity-measurement.md`](exercises/exercise-04-bias-and-toxicity-measurement.md) — run BBQ, RealToxicityPrompts with at least two classifiers, and a fairness slice on refusal-rate disparities; produce a per-slice FPR audit on AAVE / disability / LGBTQ+ language that quantifies classifier inflation and separates model behavior from classifier noise.
- [`exercise-05-prompt-injection-robustness-suite.md`](exercises/exercise-05-prompt-injection-robustness-suite.md) — run Tensor Trust (direct) and BIPIA (indirect) against a candidate model, report the utility/ASR pair per defense configuration, add InjecAgent trajectory measurement if the target is an agent, and ship the security-boundary and data-handling policies.

Reference solutions live in the paired [`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for primary references — HarmBench (Mazeika et al. 2024), XSTest (Röttger et al. 2024), WMDP (Li et al. 2024), Cybench (Zhang et al. 2024), METR's task suite, Anthropic's Responsible Scaling Policy, OpenAI's Preparedness Framework, UK AISI's published pre-deployment evaluations, BBQ (Parrish et al. 2022), RealToxicityPrompts (Gehman et al. 2020), Sap et al. 2019 on toxicity-classifier bias, BIPIA (Yi et al. 2023), InjecAgent (Zhan et al. 2024), Greshake et al. 2023 on indirect prompt injection, and the Mitchell et al. 2019 model-card format used as the Chapter 7 scaffold.
