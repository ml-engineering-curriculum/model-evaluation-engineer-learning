# mod-104-llm-benchmark-harnesses: LLM Benchmark Harnesses: lm-eval-harness, HELM, OpenAI Evals, Inspect

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Register and run a new task in EleutherAI lm-evaluation-harness end-to-end with both log-likelihood and generation evaluators
- Reproduce a public lm-eval-harness or HELM result within reported tolerance and explain any gap (prompt format, decoding, normalisation)
- Build an Inspect (UK AISI) eval with solver/scorer plumbing and a custom dataset
- Build an OpenAI evals-style registry entry with model-graded scoring
- Diagnose prompt-format sensitivity, log-prob-vs-generation gap, and per-task reproducibility (seeds, decoding params, dataset hash)

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
