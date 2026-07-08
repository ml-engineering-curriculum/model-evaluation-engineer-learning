# mod-111-eval-platform-engineering: Eval Platform Engineering: Eval-as-a-Service

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Design an eval-as-a-service architecture with a versioned task / dataset / judge / prompt registry
- Embed multiple runners (lm-eval-harness, Inspect, OpenAI evals) behind a single orchestration plane
- Apply parallelism and cost controls (token budget, runner budget, judge-tier routing) at platform scope
- Stand up an eval data warehouse with traceable result lineage (model hash, dataset hash, judge hash, prompt hash, decoding params, seed)
- Integrate eval into CI and release pipelines and define SLA / SLO for the eval system itself

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
