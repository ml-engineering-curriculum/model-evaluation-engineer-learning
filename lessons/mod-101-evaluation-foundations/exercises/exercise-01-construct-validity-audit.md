# exercise-01: Construct Validity Audit

**Estimated effort:** 2 hours

## Objective

Take a published eval design and write a construct-validity audit: identify the construct the eval claims to measure, the operationalization it uses, and the specific channels through which operationalization can move without the construct moving. Produce a memo that a colleague could use to decide whether to trust the eval for a specific downstream decision.

## Prerequisites

- Chapters 01 and 02 of this module.
- Read the primary source paper (or model-card / eval-card) for one of the evals in the target list. Do not audit from summaries.

## Target evals (pick one)

Choose exactly one from the list below. All are publicly documented and have primary-source descriptions you can read.

- **MMLU** (Hendrycks et al., 2021, "Measuring Massive Multitask Language Understanding") — construct claim: "multitask academic and professional knowledge."
- **HumanEval** (Chen et al., 2021, "Evaluating Large Language Models Trained on Code") — construct claim: "Python program synthesis from natural-language docstrings."
- **HellaSwag** (Zellers et al., 2019) — construct claim: "commonsense inference / next-event prediction."
- **HELM Core Scenarios** (Liang et al., 2022, "Holistic Evaluation of Language Models") — pick one scenario and audit its construct claim.
- **MT-Bench** (Zheng et al., 2023, "Judging LLM-as-a-Judge") — construct claim: "multi-turn instruction-following quality as scored by an LLM judge."
- **BIG-bench** (Srivastava et al., 2022) — pick one specific task and audit its construct claim.

If your organization has an internal benchmark you are allowed to write about, you may substitute it and cite the internal design doc instead.

## Requirements

Your audit must include the following sections. Total memo length: 1200–2000 words.

1. **Construct statement.** Write, in one sentence, the construct the eval's authors claim to measure. Quote or paraphrase the primary source and cite the section.
2. **Operationalization.** In 3–5 sentences, describe how the eval operationalizes that construct: item source, item format, response format, scoring rule, aggregation.
3. **Construct-validity threats.** Identify at least **four** specific threats, drawing from the taxonomy in Chapter 2 (construct underrepresentation, construct-irrelevant variance, contamination, scorer construct drift, rubric ambiguity). For each threat:
    - Name the threat.
    - Explain the mechanism in this eval, specifically.
    - Cite evidence (a paper, a blog post, a documented incident, or a first-principles argument tied to the eval's mechanics). Do not invent evidence — if you cannot cite it, mark it `<!-- needs-research: ... -->` and explain what you would need to verify.
    - Estimate severity as low / medium / high and justify.
4. **Downstream-decision framing.** Pick one specific decision the eval might be used to inform (e.g. "should we promote model B to production," "should we include model X in a purchasing shortlist," "should we cite this benchmark in a governance filing"). Argue whether the eval, given the threats you identified, is fit for that decision. If the answer is "conditionally yes," list the conditions.
5. **Recommendations.** Propose 2–4 concrete changes to the eval — additional items, a scoring change, a decontamination check, a judge calibration — that would materially reduce the top threat(s).

## Starter guidance

- Read the primary paper first, without notes. On a second pass, mark the sentences where the authors defend the construct claim (typically Section 1 or 2, and sometimes appendices). Those are the sentences you are auditing.
- The "construct-irrelevant variance" bucket is where most real-world threats live in LLM evals. Prompt-format sensitivity, answer-extraction regex brittleness, and judge biases (length, position, self-preference) all belong here.
- For contamination threats on public benchmarks, do not claim contamination "has happened" unless you have a citation showing it. It is legitimate to write "on a frontier model post-2024, contamination is plausible and undocumented; the eval report does not include a decontamination check."
- The severity rating is calibration against your downstream decision. A medium-severity threat can be a showstopper for a governance filing and irrelevant for an internal engineering decision — say so.

## Acceptance criteria

Your memo is acceptable if a reviewer can answer "yes" to every item below by reading it once:

- The construct statement is quoted or paraphrased with a specific citation, not summarised from memory.
- Each of the four (or more) threats names the specific mechanism in this eval, not a generic threat template.
- Every empirical claim (e.g. "the judge is length-biased," "this dataset overlaps with C4") has either a citation or an explicit `<!-- needs-research: ... -->` marker.
- The downstream-decision section names a specific decision, not "some decision."
- The recommendations are concrete enough that an engineer could implement one in a day of work.

## Stretch goals

- Extend the audit to a **second** eval that measures the same construct differently (e.g. audit both HumanEval and MBPP for Python program synthesis). Compare their threat profiles.
- Reproduce one threat empirically: pick a construct-irrelevant-variance channel (prompt format, decoding temperature, judge position bias) and run a small experiment showing how much the reported metric moves when only that channel changes. `n = 50` items is sufficient for a demonstration. Report the effect with a Wilson or paired-bootstrap CI (Chapters 3–4).
- Write a one-page "eval card" for the audited eval in the style of the HELM `metadata` or the Hugging Face `dataset_infos.json` schema, capturing the construct claim, the sampling frame, the known threats, and the appropriate use cases.
