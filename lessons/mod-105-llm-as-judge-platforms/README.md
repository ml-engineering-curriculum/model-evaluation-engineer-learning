# mod-105-llm-as-judge-platforms: LLM-as-Judge Platforms: Rubrics, Bias Controls, and Calibration to Humans

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 15 hours

## Learning objectives

- Design an absolute (single-answer) and a pairwise rubric for a real product task with explicit scoring anchors
- Implement position-bias controls (swap-and-average), length-bias controls (length-normalised scoring), and self-preference diagnostics
- Calibrate a judge against a human gold set (Cohen's kappa, Spearman / Kendall agreement) and decide whether the judge is good enough to ship
- Stand up an Arena-style pairwise rating system (Bradley-Terry / ELO-style) and reason about its statistical properties
- Choose between hosted (frontier API) judges, open-source judges (Prometheus, JudgeLM), and dedicated trained judges, with cost-vs-quality routing

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
