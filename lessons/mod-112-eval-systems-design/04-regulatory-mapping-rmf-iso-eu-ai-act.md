# Mapping the Eval Plan to NIST AI RMF, ISO/IEC 25059, and the EU AI Act

The third governance artifact is the regulator crosswalk. When it works, it is a translation table: on one side, the eval program's own vocabulary — datasets, judges, gates, refusal rates, jailbreak ASRs — and on the other side, the vocabulary a regulator, auditor, or compliance function uses. The audience is the compliance officer preparing a response to a regulator's request, the internal audit function preparing an ISO/IEC 42001 conformity claim, or the eval owner defending the eval program in a due-diligence exchange with an enterprise customer's legal team. The failure mode the crosswalk prevents is the *audit ambush*: the regulator or auditor asks a question in their vocabulary, the eval program has measured the underlying thing well, but the mapping has never been written down and the response is a scramble.

This chapter is a crosswalk from the eval-engineering side. It is not a legal analysis of the regulations, and it should not be treated as authoritative regulatory guidance — that lives with legal counsel and with the standards bodies' own texts. What this chapter provides is the discipline of translating what the eval program does into the frames three specific regulatory regimes expect. The three regimes were chosen because they are the ones most eval programs will encounter first: the U.S. non-binding but influential NIST AI Risk Management Framework, the technology-neutral ISO/IEC 25059 quality model, and the binding EU AI Act. Others (ISO/IEC 42001, ISO/IEC 23894, U.K. AISI evaluations, sector-specific rules under HIPAA, FDA, ECOA, and so on) follow the same crosswalk discipline; the reader who masters the three below will produce the fourth without new machinery.

## The crosswalk-as-artifact structure

Before walking each regime, it is useful to state what the crosswalk *is* as a delivered artifact. Skipping this step is how organizations end up with "we do all of this" narrative statements that a regulator or auditor cannot verify against the eval evidence.

A crosswalk row commits to five fields:

- **Regime clause.** A specific citation into the regulatory text — a section of the NIST RMF, a characteristic in ISO/IEC 25059, an article in the EU AI Act. The citation is precise (article, subsection, or characteristic name) rather than the name of the regulation.
- **What the clause asks for.** A short paraphrase, in the eval engineer's language, of what the clause is telling the organization to demonstrate. Two lines is fine; a paragraph is too much.
- **Eval-program artifact that satisfies it.** The specific dataset, evaluation run, gate, model card section, or warehouse query that constitutes the evidence. mod-111's registry and warehouse identifiers make these citations precise.
- **Traceability path.** How a reader who wants to verify the claim would move from the crosswalk row to the ground-truth evidence. Ideally a URL or a warehouse ID, not a description.
- **Gap disclosure.** If the eval program does not currently satisfy the clause, the crosswalk says so — explicitly — and names the remediation plan and the responsible owner. A crosswalk that hides gaps is worse than a crosswalk that names them; the auditor's job is to find the gaps, and a hidden gap discovered by the auditor is treated more severely than a disclosed gap with a remediation plan.

Two invariants across every crosswalk this module produces:

- **The crosswalk is versioned.** The regulations evolve. The eval program evolves. A crosswalk with a version and a change log is one an auditor can trust to represent a specific point in time; a crosswalk with no history is one whose current state is un-provable.
- **The crosswalk cites the regime by version, not by generic name.** "NIST AI RMF 1.0 (2023) plus the Generative AI Profile (NIST AI 600-1, 2024)" is a citable claim; "the NIST framework" is not. Regulatory documents get revised; the crosswalk pins the version it was written against.

The three sections below walk each regime and identify the load-bearing clauses the eval program is most likely to be asked about. The intent is not to enumerate every clause — for that, the reader consults the regime text — but to name the clauses whose crosswalk is worth the eval owner's early attention.

## NIST AI Risk Management Framework: the Measure and Manage functions

The NIST AI RMF (version 1.0, January 2023) organizes AI risk management into four functions — Govern, Map, Measure, and Manage — with subcategories that lay out what a mature program does under each. The Generative AI Profile (NIST AI 600-1, July 2024) extends the framework with generative-specific measurement expectations. The framework is non-binding in the U.S. federal-contractor sense but is treated as a de facto expectation by many enterprise customers and by U.S. federal agencies, and it shows up in due-diligence questionnaires with high frequency.

The eval program's crosswalk to the RMF concentrates on two of the four functions: Measure and Manage. Govern and Map are broader organizational functions the eval program contributes to but does not own end-to-end.

### The Measure function

The Measure function's subcategories cover the acts of quantifying identified risks — running evaluations, tracking metrics, documenting the measurement approach. Load-bearing subcategories the eval program should crosswalk explicitly:

- **MEASURE 2.3 (system performance).** Demonstrate that the AI system performs as intended and appropriate metrics have been identified. The crosswalk points at the release-gate plan (Chapter 2) and its per-gate metric taxonomy; the appropriate-metrics claim points at mod-101 Chapter 1's construct-validity documentation.
- **MEASURE 2.5 (validity and reliability).** Test the AI system's validity and reliability. The crosswalk points at the mod-101 statistical machinery (confidence intervals from Chapter 4, paired tests from Chapter 5), the mod-102 benchmark provenance work, and mod-110's re-baselining discipline.
- **MEASURE 2.7 (safety).** Evaluate for safety risks. The crosswalk points at mod-109's dangerous-capability and jailbreak measurements, the pinned attacker suites, and the safety gates from Chapter 2.
- **MEASURE 2.8 (security and resiliency).** Test for security vulnerabilities. The crosswalk points at the red-team suite from mod-109 and at the injection / prompt-attack measurements maintained there.
- **MEASURE 2.9 (explainability and interpretability).** Demonstrate the transparency of the system. The crosswalk points at the model card (Chapter 3) and at the data statements per evaluation dataset.
- **MEASURE 2.11 (fairness).** Demonstrate the fairness properties of the system. The crosswalk points at mod-103 Chapter 4's fairness measurements and at the fairness gates from Chapter 2.
- **MEASURE 3.x (tracking metrics over time).** Demonstrate that metrics are tracked over time. The crosswalk points at mod-110 Chapter 6's continuous-monitoring altitude and mod-111 Chapter 5's warehouse queries for metric-over-time.
- **MEASURE 4.x (feedback loops).** Demonstrate that measurement outcomes feed back into risk management. The crosswalk points at the incident review process invoked by rollback signals (Chapter 2) and at the change-log discipline of the plan itself.

A concrete crosswalk row against MEASURE 2.7:

```
regime_clause: NIST AI RMF 1.0 — MEASURE 2.7 (safety)
what_it_asks:  System evaluated for safety risks; identified risks are documented.
eval_artifact: mod-109 red-team suite (registry: safety-eval@rev:9a2f...);
               safety gates in release plan v2026-08-01.3;
               model card sections 4 (Metrics), 8 (Ethical considerations).
traceability:  warehouse://run/12345, plan://gates/safety.*, card://sections/8
gap_disclosure: no coverage of biological-weapons-uplift category before 2026-Q4;
                remediation planned Q1 2027 (owner: safety-eval-owner).
```

The row is short, it cites, it discloses. A crosswalk full of rows like this is the artifact that answers a due-diligence questionnaire without a scramble.

### The Manage function

The Manage function's subcategories cover the acts of prioritizing, responding to, and mitigating identified risks. Load-bearing subcategories:

- **MANAGE 1.3 (prioritizing risks).** Demonstrate that the highest-priority risks are identified. The crosswalk points at the safety-gate severity ranking (Chapter 2) and the rollback criteria (Chapter 2 second half).
- **MANAGE 2.x (planning responses).** Demonstrate that response plans exist. The crosswalk points at the rollback plan, at the incident-response runbooks, and at the compliance escalation paths.
- **MANAGE 3.x (documenting decisions).** Demonstrate that risk-related decisions are documented. The crosswalk points at the plan's change log, at overrides recorded in the warehouse, and at the model card's version history.
- **MANAGE 4.x (monitoring and updating).** Demonstrate that residual risk is monitored and the plan is updated. The crosswalk points at the sequential monitors from mod-110 Chapter 5 and at the plan-versioning discipline.

The Measure and Manage crosswalk together cover the majority of what an eval program will be asked to defend under the NIST RMF. The Generative AI Profile (NIST AI 600-1) adds twelve categories of risk specific to generative systems (confabulation, dangerous or violent recommendations, environmental impact, and so on) that each map to specific safety evaluations from mod-109; the crosswalk from the Profile to the mod-109 suite is left as an exercise for the reader and forms part of exercise-03.

## ISO/IEC 25059: quality characteristics for AI systems

ISO/IEC 25059:2023 is the AI-specific extension of the SQuaRE (Systems and software Quality Requirements and Evaluation) family (ISO/IEC 25010 and related). Where NIST speaks in terms of risk management functions and the EU AI Act speaks in terms of obligations, ISO/IEC 25059 speaks in terms of *quality characteristics*: named attributes of an AI system that quality evaluation should address. It is the vocabulary organizations use when they need a technology-neutral quality claim — a claim that neither commits to any specific regulator's frame nor invents its own — and it composes especially well with ISO/IEC 42001, the AI management-system standard.

The crosswalk to 25059 has a different shape from the NIST crosswalk: each row maps a quality characteristic (or sub-characteristic) to the eval-program artifact that measures it. The characteristics span both the base 25010 quality model (already applicable to any software system) and the 25059 AI-specific additions.

Load-bearing characteristics the eval program's crosswalk should cover:

- **Functional suitability** (functional completeness, functional correctness, functional appropriateness). Crosswalk to the functional-quality gates from Chapter 2 and the mod-103 task-level accuracy measurements.
- **Performance efficiency** (time behavior, resource utilization, capacity). Crosswalk to the latency gates (TTFT, TPOT, throughput) from Chapter 2 and to the mod-110 Chapter 7 serving-benchmark discipline.
- **Reliability** (maturity, availability, fault tolerance, recoverability). Crosswalk to mod-110 Chapter 6's drift alerts and mod-111 Chapter 6's platform SLOs on availability.
- **Security** (confidentiality, integrity, non-repudiation, accountability, authenticity, resistance). Crosswalk to mod-109's prompt-injection and adversarial suites and to the mod-111 Chapter 5 data-privacy handling of eval payloads.
- **Usability.** Crosswalk to the model card's intended-use disclosure and to any downstream integrator-facing documentation.
- **Compatibility** (co-existence, interoperability). Crosswalk to the model card's deployment-context disclosure and to the mod-111 Chapter 2 registry's runner-compatibility fields.
- **Maintainability.** Crosswalk to the plan-versioning discipline and to the eval platform's own change-management discipline.
- **Portability.** Usually less load-bearing for hosted-model programs; crosswalk to any evidence about deployment on multiple substrates when applicable.
- **AI-specific: functional adaptability.** The AI-specific characteristic covering how well the system performs across the range of inputs it is expected to handle. Crosswalk to the mod-102 gold-set coverage and to slice-level breakdowns from mod-103.
- **AI-specific: user controllability.** How well the user can control the AI system's behavior. Crosswalk to the mod-109 over-refusal measurements and to any documented user-configuration surface.
- **AI-specific: transparency.** How well the AI system's behavior is explainable. Crosswalk to the model card and to any interpretability evidence the program maintains.
- **AI-specific: robustness.** How well the system copes with disturbances. Crosswalk to the mod-109 adversarial suite and to the mod-110 shadow / A/B measurements under production-drift conditions.
- **AI-specific: intervenability.** How well the system supports human intervention. Crosswalk to the rollback plan (Chapter 2 second half), the override records in the warehouse, and any human-in-the-loop escalation paths documented in the deployment.

Crosswalk row shape (against 25059 functional appropriateness):

```
regime_clause: ISO/IEC 25059:2023 — functional appropriateness (from 25010)
what_it_asks:  System functions facilitate accomplishment of the specified tasks and objectives.
eval_artifact: mod-101 Chapter 1 construct-validity documentation for the internal judge;
               mod-102 gold set for the intended-task inventory (registry: gold-set@rev:c1d2...);
               task-level accuracy gates in release plan v2026-08-01.3.
traceability:  card://sections/2, plan://gates/quality.*, registry://datasets/gold-set@rev:c1d2
gap_disclosure: none.
```

ISO/IEC 25059 is the crosswalk regime most likely to be requested by a customer performing procurement due diligence in a regulated industry outside the EU; NIST is common in the U.S., EU AI Act is legally binding in the EU, and 25059 fills the space in between.

## The EU AI Act: high-risk and GPAI obligations

The EU AI Act (Regulation (EU) 2024/1689) is binding law in the EU. It classifies AI systems into unacceptable-risk (prohibited), high-risk (heavily regulated), limited-risk (transparency obligations), and minimal-risk categories; it also separately regulates general-purpose AI (GPAI) models, with additional obligations for GPAI models with systemic risk. Most eval programs will encounter it in one of two shapes: a system whose deployment context puts it into the high-risk category under Annex III (uses like recruitment, credit scoring, education), or a GPAI model that is placed on the EU market.

This section is the eval crosswalk to the two obligations that most directly involve evaluation. It is not a legal analysis. Legal counsel names the classifications, the obligations that follow, and the evidence sufficient to demonstrate conformity; the eval program's job is to produce evidence that legal counsel can hand over.

### High-risk systems: Article 15 (accuracy, robustness, and cybersecurity)

Article 15 requires that high-risk AI systems be designed and developed to achieve appropriate levels of accuracy, robustness, and cybersecurity, and that they perform consistently in those respects throughout their lifecycle. The Article names the three properties and requires disclosure of their metrics in the instructions for use.

Crosswalk from the eval program:

- **Accuracy.** The measured task-level accuracy on the intended-use tasks, with confidence intervals, sample sizes, and per-slice breakdowns. Sources: mod-103 slice metrics, mod-101 confidence-interval discipline, Chapter 2's functional-quality gates. Article 15's requirement to disclose accuracy metrics in the instructions for use is what makes the model card's metric section a compliance artifact for EU-market high-risk deployments.
- **Robustness.** Measured performance under adversarial inputs, distribution shift, and long-tail conditions. Sources: mod-109's adversarial suites, mod-110 Chapter 3's shadow-under-non-IID-traffic discipline, and the mod-108 agent-tool-eval work for agentic systems.
- **Cybersecurity.** Resistance to attempts to manipulate the model or exfiltrate confidential data. Sources: mod-109's prompt-injection suites, the eval platform's own access-control and data-handling posture (mod-111 Chapter 5), and any documented security review of the deployment.

Article 15 also requires "measures to prevent or mitigate any inaccurate, non-robust, or non-secure behavior." The crosswalk to those measures is the release-gate plan's action-and-rollback discipline: gates that block on regressions in these dimensions are the preventive measure, and the rollback plan is the mitigating measure.

Crosswalk row shape (against Article 15):

```
regime_clause: EU AI Act — Article 15(1) (accuracy, robustness, cybersecurity)
what_it_asks:  High-risk AI system achieves appropriate accuracy, robustness, cybersecurity;
               performs consistently in these respects across its lifecycle.
eval_artifact: model card sections 4 (metrics) and 7 (quantitative analyses);
               mod-109 adversarial suite (registry: adv-suite@rev:5f9c...);
               mod-110 shadow deployment results (warehouse://run/23456);
               release plan v2026-08-01.3 (functional, safety, latency gates and rollback).
traceability:  card://sections/4-7, plan://gates and plan://rollback, warehouse://
gap_disclosure: cybersecurity evidence pre-2026 does not cover the multi-agent
                tool-use attack surface; remediation Q4 2026 (owner: red-team lead).
```

### High-risk systems: Annex IV (technical documentation)

Annex IV of the EU AI Act enumerates the technical documentation that must be prepared for high-risk AI systems and made available to authorities on request. It is a substantial list; a full crosswalk lives in the exercise-03 deliverable. The high-frequency items the eval program owns evidence for:

- **General description and intended purpose.** Sources: the model card's intended-use section and the release-gate plan's intended-use section (kept in sync per Chapter 3).
- **Detailed description of the elements of the AI system and the process for its development.** Sources: partially from the eval program (evaluation methods, judges, datasets), partially from the model-development team.
- **Detailed information about monitoring, functioning, and control.** Sources: the release-gate plan, the rollback criteria, mod-110 Chapters 5 and 6 on monitoring, and the mod-111 platform SLOs.
- **Description of the appropriateness of performance metrics.** Sources: mod-101 Chapter 1's construct-validity documentation, keyed to each metric in the model card.
- **Detailed description of the risk-management system.** Sources: the incident-response runbooks, the plan's change log, and the crosswalk artifact itself (which demonstrates that the risk-management system is aware of these obligations).
- **Description of the training methodologies and techniques, including datasets used.** Sources: partially the model-development team; partially the mod-102 provenance documentation for the eval datasets that share provenance with training corpora, insofar as contamination assessments are part of the disclosure.
- **Evaluation of the AI system.** Sources: the model card's metrics and quantitative-analyses sections, the release-gate plan's gate results, and warehouse queries for the specific runs.
- **Cybersecurity measures adopted.** Sources: the mod-109 injection and adversarial suites, plus the platform-security posture from mod-111.
- **Post-market monitoring plan.** Sources: mod-110 Chapters 5 and 6 continuous monitoring, plus the incident-response and rollback criteria.

Every Annex IV item resolves to specific eval artifacts. The crosswalk artifact makes those resolutions explicit; without it, the compliance function is doing the resolution ad hoc every time a documentation request comes in.

### GPAI: Article 55 (obligations for models with systemic risk)

Article 55 covers general-purpose AI models with systemic risk — a classification triggered by capability thresholds (currently benchmarked against training FLOPs, subject to update by the Commission). Obligations include performing model evaluation with state-of-the-art protocols and tools, assessing and mitigating systemic risks, tracking and reporting serious incidents, and ensuring adequate cybersecurity.

Crosswalk from the eval program:

- **Model evaluation with state-of-the-art tools.** Sources: mod-104's benchmark-harness discipline, mod-109's dangerous-capability and red-team suites, the mod-111 platform for reproducibility. The point in the crosswalk is not that a specific benchmark was run; it is that the eval program tracks the state of the art (which is a mod-111 registry update discipline) and adopts protocols with justifications.
- **Systemic-risk assessment.** Sources: mod-109's dangerous-capability evaluations for the categories that constitute systemic risk (biological, chemical, cyber, autonomy), and the crosswalk of those categories to the NIST AI 600-1 GAI Profile's risk taxonomy.
- **Incident reporting.** Sources: the mod-110 sequential-monitor alerting plus the incident-response runbook. The eval program owns the *detection* side; the reporting-to-authorities side is legal and comms.
- **Cybersecurity.** As Article 15, above.

GPAI systemic-risk classification is a moving target and legal counsel's determination; the eval program's job is to have evidence ready that supports (or contests, if the classification is wrong) the assessment.

## Composing the three crosswalks

The three crosswalks share a substrate — the eval program's own artifacts — and diverge in vocabulary. A crosswalk repository has one source of truth (the artifacts themselves, in mod-111 registry and warehouse) and multiple *views* over it, one per regime.

Two disciplines keep the three coherent:

- **Every artifact declares its regime tags.** The mod-111 registry stores, alongside each artifact (dataset, judge, gate, run), a `regime_tags` field listing the regimes and clauses the artifact serves as evidence for. `{regime: "eu-ai-act", clauses: ["art 15", "annex IV item 4"]}` on a specific dataset revision means that the crosswalk views for the EU AI Act can be generated by query rather than hand-maintained.
- **The crosswalks are generated views, not hand-maintained tables.** A build step queries the registry, gathers the `regime_tags` and their citations, and produces the per-regime crosswalk artifact. A hand-maintained crosswalk drifts; a generated crosswalk stays current on every registry commit.

The generated-crosswalk pattern is not universally practiced yet; many organizations maintain crosswalks as spreadsheets. The spreadsheet approach works when the eval program is small and the regime coverage is stable, and it breaks down as either grows. The organization that expects to be under active regulatory scrutiny is well-served by the generated-crosswalk pattern from the start.

## When the eval program does not satisfy a clause

The crosswalk's gap-disclosure field is the discipline that makes the artifact honest when the eval program does not currently do everything a clause asks for. Three properties keep gap disclosures useful.

- **The gap is specific.** "We do not measure X on population Y before deadline Z" is a useful gap. "Robustness measurement is a work in progress" is not.
- **The remediation is committed.** A specific plan with an owner and a date. A gap without a plan is a red flag; a gap with a plan is a project.
- **The gap is visible in the crosswalk itself.** Not hidden in a footnote or a separate document. A crosswalk that hides gaps is one the auditor will treat as adversarial when they find them.

An eval program with no gaps is either extraordinary or dishonest. A crosswalk that discloses two gaps with active remediation plans is more credible than one that discloses none.

## Guidance for the crosswalk author

- **The crosswalk is a translation, not a claim of compliance.** The eval program is not the entity that declares itself compliant; the compliance function and legal counsel do. The crosswalk provides the evidence they consume.
- **Cite the regime by version, and the clause by number.** "The EU AI Act" is not a citation; "Regulation (EU) 2024/1689, Article 15(1)" is.
- **Generate the crosswalk from the mod-111 registry when you can.** Hand-maintained crosswalks drift on every eval-program change; generated crosswalks stay current.
- **Disclose gaps.** Hidden gaps are the failure mode audits are structured to find. Disclosed gaps with plans are what mature programs demonstrate.
- **The crosswalk composes with the model card.** The card's sections resolve into crosswalk rows; the crosswalk citations resolve into card sections. Keep them mutually reachable in the same tooling.
- **Update the crosswalk when the regime changes.** Regulations are versioned. When the EU AI Act delegated acts drop, when the NIST RMF issues a new profile, when a new 25059 amendment is published, the crosswalk versions in response — the discipline of the previous chapters applied to this artifact.

## Summary

The regulator crosswalk is the third governance artifact this module builds: a per-regime view of the eval program's evidence, structured against a specific regulatory or standards vocabulary. Three regimes cover most eval programs' first encounters: the NIST AI Risk Management Framework (1.0 plus the Generative AI Profile), whose Measure and Manage functions the eval program directly satisfies; ISO/IEC 25059's AI-specific quality characteristics, which the eval program's dataset, gate, and monitoring evidence maps to; and the EU AI Act, whose Article 15 accuracy / robustness / cybersecurity obligations, Annex IV technical documentation list, and Article 55 GPAI obligations pull specific artifacts from the plan, the card, and the mod-111 substrate. The crosswalk is versioned, structured as evidence rows with named gaps, cited by regime version and clause number, and best generated from the mod-111 registry's per-artifact regime tags rather than hand-maintained. The audience is the compliance function and — through them — the auditor or regulator; the failure mode it prevents is the audit ambush where the eval program measured the right things under a vocabulary no one translated. The next chapter turns to the fourth artifact — the build-vs-buy platform dossier — where the eval program's substrate itself becomes the object of a leadership decision.
