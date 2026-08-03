# exercise-05: Eval Report Red-Flag Review

**Estimated effort:** 2 hours

## Objective

Apply the Chapter 7 protocol — four questions, twelve-item checklist — to a real eval report and produce a reviewer memo of the kind you would send to the author before signing off on a release. The output should be short, specific, actionable, and honest about what you can and cannot conclude from the report as written.

## Prerequisites

- All previous chapters of this module (01–07).
- Access to one of the reports listed below (or an internal one you are permitted to review).

## Requirements

### Part A — pick a report

Choose one:

- **Model card, technical report, or system card** for a released model. Recent frontier-model releases publish substantial eval sections; pick one and cite the published document. Examples of the kind of document to look for (not a recommendation): Anthropic Claude model cards, OpenAI system cards, Google Gemini technical reports, Meta Llama model cards, Mistral or Qwen tech reports, Cohere Command model cards. Choose a document with substantive eval content (multiple benchmarks, human eval, safety eval), not a marketing page.
- **A published benchmark paper** where the authors present results comparing multiple systems (e.g. the results section of the HELM paper, the MT-Bench paper, or a BIG-bench-style multi-task paper).
- **An internal report** at your organization, if you have permission to review and write about it. In this case cite the internal doc rather than a URL.

Do not use a report you helped author.

### Part B — the 12-item checklist

Score the report on each of the 12 items in Chapter 7 as `yes` / `no` / `n/a`. For every `no`, write one sentence naming the specific gap ("no CI reported on the headline MMLU number"; "prompt template not documented for the challenger models"; "per-language slice claims made without multiplicity correction across 26 languages").

### Part C — the four-question walkthrough

For **one specific claim** in the report — pick the one that carries the most decision weight, e.g. the headline capability improvement or the "safety refusal rate" claim — write out the four-question analysis:

1. **Validity.** What construct does the claim invoke, and what is the operationalization? What are the top two construct-validity threats?
2. **Sampling.** Where does the eval set come from? How well does it match the deployment surface? What does the report say (or fail to say) about the sampling frame?
3. **Statistical power.** Given the reported sample size and (if reported) agreement structure, is the eval powered to detect the claimed effect? Use the MDE table in Chapter 5.
4. **Contamination.** If the eval is a public benchmark, does the report include a decontamination check? If yes, describe the method and comment. If no, state the plausibility of pretraining exposure.

### Part D — the reviewer memo

Compose a single memo (600–1000 words) that a colleague could use to decide whether the report supports its top-line recommendation. It must contain:

- One paragraph naming the top three issues, in order of severity, with a specific fix for each ("promote model B" ← ask for BH-corrected per-language slice results; "safety refusal rate improved" ← ask for the sample size on the safety eval and a CI).
- A paragraph on what the report does well. Not everything is a red flag; identify at least two things the report handles competently and say why.
- A decision recommendation: "accept," "accept with minor revisions," "revise and resubmit," "reject as basis for the claimed decision." Justify the recommendation in one paragraph.
- A "questions for the author" section: 3–5 specific questions whose answers would resolve the top-severity issues.

The memo should not moralize about eval hygiene in general; it should critique this specific report on specific claims.

## Starter guidance

- Read the report once, cover-to-cover, without notes. On a second pass, apply the checklist. On a third, write the walkthrough.
- Model cards and system cards are the easiest starting point because they concentrate a lot of eval reporting into one document. They are also often the *worst* on the checklist — model cards are marketing documents as much as technical ones. That contrast is instructive.
- Do not fabricate a claim in the report to critique it. If a report is silent on X, the review is "the report is silent on X and this is a gap for [reason]" — not "the report incorrectly asserts X."
- When you invoke evidence — "prompt-format sensitivity moves MMLU scores by several points," "LLM judges have documented length bias" — cite the paper. `<!-- needs-research: ... -->` if you cannot.
- Severity is calibrated against the decision. A missing prompt template is a fatal issue for reproducibility, a moderate issue for a "which model is better" decision on stable benchmarks, and a minor issue for a "should we experiment further" decision.

## Acceptance criteria

Your submission is acceptable if:

- The report you review is named with a citation or link.
- All 12 checklist items are scored `yes` / `no` / `n/a` with a specific one-line reason for each `no`.
- The four-question walkthrough is applied to one identifiable claim, not the whole report abstracted.
- The memo has all four sections (top-three issues, what the report does well, decision recommendation, questions for author).
- Every claim you make about the report is defensible from the report's own text; every claim you make about the literature is cited or marked `<!-- needs-research: ... -->`.
- The memo is written to be sent — no throat-clearing, no self-referential ("in this exercise I will...") language.

## Stretch goals

- Repeat the analysis on **two** reports evaluating the same claim (e.g. two labs' model cards for models compared on the same benchmark) and compare their eval-hygiene profiles. Which methodological choices differ, and does the divergence explain any of the reported score differences?
- **Author the improved eval report** the reviewed report should have shipped: same claims, but with the CIs, decontamination discussion, prompt templates, and multiplicity corrections filled in. Where the original report is silent on a fact you would need, use `<!-- needs-research: ... -->` rather than fabricating.
- **Publish the review** (with permission) on a technical blog or as an issue in the corresponding public repo. Post-review discussion with the authors is often the fastest way to learn what a report is actually claiming.
