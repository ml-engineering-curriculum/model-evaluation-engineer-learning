# Authoring the Safety Section of a Model Card

Chapters 2–6 built six measurements: refusal, over-refusal, jailbreak ASR (per attacker), dangerous-capability domain scores (mapped to public risk frameworks), bias / toxicity / fairness, and prompt-injection robustness. The last chapter of this module is about assembling those measurements into a single artifact that the model card, the safety board minutes, and the external auditor's report all reference. That artifact has an unusual constraint: it must satisfy governance and risk partners — who need enough detail to make judgements — without leaking the attack payloads that produced the numbers.

The chapter is opinionated about *format*. It draws heavily on the shape used by the GPT-4 System Card, the Anthropic Claude model cards, the Llama 3 / 3.1 model cards, the Gemini technical report, and the model-card format proposed by Mitchell et al. 2019 ("Model Cards for Model Reporting"). None of those artifacts are identical, but they converge on the same rules for what belongs in the external write-up versus what stays internal.

## Two audiences, two artifacts

A safety report is not a single document. It is at least two:

- **The public model card / system card.** Circulates externally: on the model provider's website, to enterprise customers under NDA, to regulators, to the press. Contains aggregate metrics, methodology descriptions, framework mappings, known limitations, mitigation state. Does *not* contain raw attack payloads, harmful completions, or per-item verdicts on the harmful side.
- **The internal safety evaluation report.** Circulates within the organization, to the safety board, to red-team collaborators under confidentiality, and to independent evaluators like UK AISI under contract. Contains everything the public artifact contains *plus* per-item verdicts, sampled attack examples (redacted), detailed judge-rationale traces, and the raw data-handling audit.

Chapter 1 drew the boundary: awareness in the external artifact, controlled detail in the internal. This chapter walks the external side — the shape that must not leak payloads — and describes the internal side by reference (what belongs there without spelling out the contents).

## The public safety-section skeleton

A defensible safety section of a model card has six subsections. The exact wording is style-dependent; the sections are load-bearing.

### 1. Policy statement

What the model is designed to refuse and to help with, at a category level. Not a legal document (that lives in usage policies and terms of service); a short statement of the *evaluation policy* — the ground truth against which "refused correctly" or "did not refuse correctly" is judged.

Example (paraphrased):

> This model is evaluated under Policy v2.4 (dated 2026-06-15), which requires refusal on requests for weapons synthesis, targeted harassment content, sexual content involving minors, credential theft instructions, and self-harm facilitation. It requires assistance on medical, legal, security, chemistry, and dark-humor questions when the request is not itself an attempt to cause harm. Full policy: `<link>` (available to enterprise customers under NDA).

The policy statement fixes the ground truth for the refusal / over-refusal / jailbreak numbers below. Without it, those numbers do not have a defined success criterion.

### 2. Refusal and over-refusal

The Chapter 2 measurements. Two numbers with CIs, per-category slicing, comparison against prior release. Never composite.

Example table:

```
Refusal rate on harmful set (n=520, HarmBench standard split + private held-out, rev abc123):
  Overall:                       0.94  [0.92, 0.96]
  Weapons:                       0.99  [0.97, 1.00]
  Self-harm:                     0.97  [0.94, 0.99]
  Credential theft:              0.92  [0.87, 0.96]
  ...

Over-refusal rate on benign-lookalike set (n=250, XSTest + private, rev def456):
  Overall:                       0.11  [0.07, 0.15]
  Medical questions:             0.08
  Security questions:            0.15
  Dark-humor / hypotheticals:    0.14
  ...

Compliance judge: Llama-Guard-3-8b, agreement with human gold: 91% (harmful), 87% (benign).
```

### 3. Jailbreak resistance

The Chapter 3 measurements. ASR per attacker profile, per category, with the attacker specification.

Example table:

```
Jailbreak resistance (HarmBench standard split, rev abc123, judge: HarmBench classifier v2):

By attacker profile:
  static-baseline (k=1):              ASR 0.02  [0.01, 0.04]
  single-turn transformations (k=1):  ASR 0.07  [0.05, 0.10]
  PAIR (k=20):                        ASR 0.19  [0.15, 0.23]
  TAP (k=20):                         ASR 0.24  [0.20, 0.28]

Multi-turn (Crescendo, 8-turn, attacker-in-the-loop, k=1): ASR 0.31  [0.26, 0.37]
```

*Do not* include per-behavior examples of successful attacks in this table. The category-level breakdown (illegal, cybercrime, chem/bio, etc.) is safe; per-behavior specifics are not.

### 4. Dangerous-capability evaluation

The Chapter 4 measurements. Per domain, per benchmark, with the framework mapping and the human-review pointer.

Example paragraph shape:

> **Cyber offense.** Evaluated on Cybench Verified (rev xyz789) with a documented agent harness (Inspect + `basic_agent` at 40-step budget); task-solve rate 0.32 [0.27, 0.37], per-difficulty breakdown in appendix. Under the Anthropic RSP v2.0 mapping this places the model well below the ASL-3 threshold for "can meaningfully assist a non-expert in offensive cyber operations." Under the OpenAI Preparedness Framework v1 taxonomy this would map to a "Low" cyber score. Governance review by the Safety Board on 2026-07-22 recorded no threshold concerns.
>
> **Autonomy.** Evaluated on METR task suite (rev pqr456), METR "long-horizon" split, agent harness identical to above; solve rate 0.24 [0.19, 0.29]. Well below RSP v2.0 ASL-3 "autonomous-replication" thresholds per internal review 2026-07-22. Cross-referenced with SWE-bench-Verified score (from the capability section) of 0.51 as a lower-bound realistic-workload data point.
>
> **CBRN / Biosecurity.** WMDP-bio 0.62 [0.59, 0.66], WMDP-chem 0.58, WMDP-cyber 0.54. Interpretation: proxy signal for hazardous-knowledge recall; not a direct measure of uplift capability. Cross-referenced with an internal biosecurity uplift study (Study 2026-Q2, n=40 participants, report available under NDA); the study found no statistically-significant uplift over web-search baseline. Framework mapping: below the RSP v2.0 "meaningfully increases risk of a bioweapons attack by non-experts" threshold per internal review 2026-07-22.

Notice: the report does not disclose the individual questions, does not disclose the model's completions on hazardous prompts, and does not include the uplift-study protocol in the public artifact. It cites the study, points at where the details live (under NDA), and lets the aggregate result inform the framework mapping.

### 5. Bias, toxicity, fairness

The Chapter 5 measurements. Per-benchmark, per-category, with FDR-corrected significance and classifier-calibration text.

Example paragraph (see Chapter 5's minimum-viable report for a fuller table):

> **BBQ (n=58k, all 11 categories, BH-adjusted at q<0.05).** In ambiguous items, three categories show statistically-significant stereotype-consistent bias: race/ethnicity (+8.9 [+6.2, +11.5]), nationality (+3.2), religion (+2.7). Disambiguated-item bias is within noise across all categories. Overall accuracy (disambig): 0.91.
>
> **Toxicity (RealToxicityPrompts, n=100 completions/prompt, 10k prompts).** Detoxify unbiased v0.5 classifier (human-gold calibration on 400 items: 87% agreement; known false-positive elevation on AAVE per Sap et al. 2019; secondary check with Perspective API v6). EMT on toxic-prompt split 0.31 [0.29, 0.33]; toxicity probability (T=0.5) on non-toxic split 0.024 [0.020, 0.028].
>
> **Fairness (refusal-rate, sliced by demographic target on n=520 harmful prompts).** No statistically-significant refusal-rate disparities at q<0.05 across the tested demographic axes. Task-performance disparity by demographic remains an open evaluation gap.

### 6. Prompt-injection robustness

The Chapter 6 measurements. Utility-and-ASR pair per defense configuration, with attack taxonomy cited.

Example:

```
Prompt-injection robustness (BIPIA v1.0, 5 task types, 25 attack methods, rev stu321):

Baseline (no application-side defense):
  Benign utility (aggregate task metric):   0.42  [0.40, 0.44]
  Injection ASR (aggregate over methods):   0.31  [0.28, 0.34]

With built-in Structured Query defense (input-tagged; see application notes):
  Benign utility:  0.40  [0.38, 0.42]  (-2 pt)
  Injection ASR:   0.14  [0.11, 0.17]  (-17 pt)

Agent tool-poisoning (InjecAgent v1.0, rev vwx654, mod-108 trajectory scorer):
  Attacker-tool-invocation rate: 0.09  [0.06, 0.13]
  Legitimate-task completion:    0.71  [0.66, 0.76]
```

### 7. Limitations and known gaps

Every safety section names what it did *not* measure. Governance partners consuming the artifact use this section to scope their own follow-on work.

Example:

> **Multi-turn adversarial testing.** Multi-turn attacks (Crescendo-style) were evaluated at 8 turns with one automated attacker; longer horizons (>15 turns) and human-in-the-loop multi-turn attacks are not covered.
>
> **Non-English coverage.** Refusal, over-refusal, and jailbreak evals were run in English (and Spanish, French, German for the top-line numbers). Coverage of low-resource languages is limited to a small canary set (n=200) per language.
>
> **Novel biosecurity uplift.** The internal uplift study (Study 2026-Q2) covered scenarios developed by the CBRN policy team and reviewed by domain experts; novel attack scenarios developed after 2026-Q1 are not covered.
>
> **Task-performance fairness.** Not measured; refusal-rate fairness is a proxy but not a substitute.

## What NEVER appears in the public artifact

Naming the exclusion list is as important as naming the inclusion list.

- **Raw attack payloads.** No jailbreak prompts, no injection strings, no HarmBench behavior descriptions verbatim beyond category names.
- **Raw model completions on the harmful side.** No sampled outputs from successful jailbreaks. No sampled outputs on dangerous-capability items where the completion carries hazardous information.
- **Per-item verdicts on the harmful side.** Category totals are fine; per-behavior verdicts (with behavior text) are not — they are a curated hit-list for the next attacker.
- **Novel exploits or vulnerabilities against specific deployed applications.** Those go through the vulnerability-disclosure channel, not the model card.
- **Attacker-model implementation details that could be reused against other models.** Cite PAIR / TAP by paper reference; do not include the attacker prompts and configuration verbatim.
- **Judge-rationale traces that quote payloads verbatim.** Judge rationales that reference "the attempted attack was of type X" are safe; rationales that quote the attack string are not.

## The internal appendix: what lives on the controlled side

For everything excluded above, the internal artifact carries the details. A defensible internal appendix has:

- **Per-item verdict tables** for all benchmarks, with `(behavior_id, attacker_id, verdict, latency, tokens)` but *not* the raw prompts or completions. The raw content lives in a separate storage tier with access-log audit.
- **Sampled attack traces** for each attacker profile, redacted for external sharing but full-fidelity internally. Retention is time-limited; access is logged.
- **Judge-calibration datasets** (human-gold labels used to calibrate the compliance and toxicity classifiers).
- **The uplift-study protocol and per-participant results** for CBRN and persuasion, under IRB-adjacent review.
- **The reproducibility manifest** for each eval run: model version, harness version, dataset revision, judge version, budget caps. This is the mod-102 discipline applied to safety.

An external auditor (UK AISI, US AISI Consortium, an enterprise customer's compliance team) engages with the internal artifact under NDA and produces their own summary; the public artifact does not need to satisfy the auditor unaided.

## Common failure modes in safety-section drafting

Three ways a well-intentioned drafter still ships a bad safety section.

**Failure mode 1: single-scalar reporting.** "Safety score: 92%." This section fails the Chapter 1 dashboard test — you have compressed six measurements into one number and lost the ability to reason about tradeoffs. Fix: replace with the six-section skeleton.

**Failure mode 2: cherry-picked attacker profile.** Reporting HarmBench static-attacker ASR of 2% as "jailbreak resistance ~2%" without mentioning that the same model has 24% ASR under TAP. Fix: report multiple attackers; publish the ASR curve.

**Failure mode 3: missing framework mapping.** Reporting WMDP-bio at 62% without saying whether that maps to ASL-3, Preparedness "Low," or something else. The number is meaningless to a governance partner without the mapping. Fix: cite the framework version and the mapping paragraph.

**Failure mode 4: leaked payloads in an appendix.** An "example jailbreak" screenshot in an appendix, a sampled model output that shows harmful content, or a table of "top 10 attack prompts we defended against." Fix: category-level in the public artifact; controlled-store for the specifics.

**Failure mode 5: no policy statement.** Numbers without a policy have no ground truth. A "refusal rate of 94%" against an unstated policy could mean anything from "94% refusal on a policy that says refuse cooking questions" to "94% refusal on weapons synthesis." Fix: publish the evaluation policy at least at category granularity.

## Answering governance-partner questions from the model card alone

The test of a well-authored safety section is whether it lets a governance partner (safety board member, compliance reviewer, external auditor, regulator staff) answer their common questions without needing the internal artifact.

A short catalog of questions the card should be able to answer:

- "What does the model refuse and what does it help with?" → Policy statement + refusal / over-refusal tables + per-category slicing.
- "How resistant is the model to a determined attacker?" → Jailbreak ASR curves per attacker profile.
- "Is this model close to a capability threshold under our framework?" → Dangerous-capability section with the framework mapping and the governance-decision pointer.
- "Are there systematic biases we need to know about?" → Bias / toxicity / fairness section with FDR-corrected findings and classifier-limitation text.
- "For deployments where the model reads untrusted content, how vulnerable is it?" → Prompt-injection section with the utility / ASR pair.
- "What was NOT evaluated that we should be worried about?" → Limitations section.

If a governance partner has to email you and ask, the section is missing something.

## Guidance for the drafter

- **Follow the six-section skeleton.** Policy, refusal/over-refusal, jailbreak, dangerous-capability, bias/toxicity/fairness, injection, plus limitations. Do not compress.
- **Publish the ingredients, not the outputs.** Benchmark names + versions, judge names + calibration numbers, attacker profiles + budgets, framework versions. A reader must be able to reason about how each number was produced.
- **Cite framework versions.** Anthropic RSP v2.0, OpenAI Preparedness Framework v1.0, UK AISI evaluation protocol v-something. Framework text and thresholds change; a mapping is only interpretable next to the specific version.
- **Name your limitations.** Sections 7 always exists. A safety section without a limitations section is over-claiming coverage.
- **Reference the governance decision.** The safety-section reports the eval results and the framework mapping; the *decision* to ship (or not) is recorded elsewhere (safety-board minutes, RSO signoff), and the section points at where that decision lives.
- **Coordinate the draft with policy, legal, security, and communications.** The safety section is a governance artifact, not an engineering artifact. Draft it with the partners who consume it, not at them.
- **Never leak payloads.** Aggregate metrics, category names, redacted truncated examples with `[TRUNCATED]` markers. Payloads and per-item verdicts on the harmful side go to the internal appendix under access control.

## Summary

The safety section of a model card assembles the six measurements from Chapters 2–6 into a single public artifact that governance and risk partners can act on without receiving raw attack payloads. Its skeleton is fixed: policy statement, refusal / over-refusal, jailbreak ASR, dangerous-capability with framework mapping, bias / toxicity / fairness, prompt-injection robustness, and a limitations section. Everything reports aggregate metrics with confidence intervals, cites the instruments and their versions, and points at the internal appendix (under access control) for the per-item detail. The exclusion list is load-bearing: no raw payloads, no harmful completions, no per-behavior hit-lists, no novel exploit disclosures, no attacker-prompt verbatim. The internal appendix carries the specifics under retention and access-audit discipline. The test of a well-authored section is whether a governance partner can answer their standard questions from the card alone; if they have to email you, the section is missing something. The module ends here; the exercises step through the six measurements as running builds and end at the model-card write-up.
