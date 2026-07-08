# mod-108-agent-and-tool-eval: Agent and Tool-Use Evaluation: Trajectories, Sandboxes, and Partial Credit

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 15 hours

## Learning objectives

- Build a trajectory-level scorer (per-step tool-call correctness, final-answer correctness, cost / latency / step-count budgets)
- Run SWE-bench (and SWE-bench-Lite / Verified) against a candidate model and reproduce the published number within tolerance
- Run a WebArena / GAIA / AgentBench task and reason about sandbox isolation and deterministic replay
- Design a partial-credit rubric for free-form trajectories and audit it against human gold
- Use Inspect's agent harness to wire a custom tool-use eval end-to-end

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
