# mod-108-agent-and-tool-eval: Agent and Tool-Use Evaluation: Trajectories, Sandboxes, and Partial Credit

**Estimated effort:** 15 hours

Every module up to this point has evaluated a single request-response pair — even in mod-107, where the model *generated code* that a sandbox ran, the eval was one turn wide: prompt in, program out, tests grade the program. That shape stops fitting the moment the model is allowed to *loop* — to inspect its own output, call a tool, observe a result, and decide what to do next. This module is about evaluating models under that shape: not just "did the final message match the target," but *what did the agent actually do*, *what did it cost*, *did the sandbox even give it a fair environment*, and *can I re-run this eval next week and get the same number*.

The chapters walk five topics in order: the trajectory-scoring vector that replaces single-turn accuracy (Chapter 2), the SWE-bench reproduction pipeline that anchors the module against a public leaderboard (Chapter 3), the interactive-sandbox benchmarks (WebArena, GAIA, AgentBench) where the environment itself is stateful and non-deterministic (Chapter 4), the partial-credit rubric that lets an LLM judge score free-form trajectories with the calibration discipline from mod-105 (Chapter 5), and Inspect's agent harness as a reference implementation that stitches all of the above together (Chapter 6). Chapter 1 sets up why single-turn scoring breaks and why this module exists as its own thing rather than being folded into mod-104 or mod-107.

## Learning objectives

- Build a trajectory-level scorer (per-step tool-call correctness, final-answer correctness, cost / latency / step-count budgets).
- Run SWE-bench (and SWE-bench Lite / Verified) against a candidate model and reproduce the published number within tolerance.
- Run a WebArena / GAIA / AgentBench task and reason about sandbox isolation and deterministic replay.
- Design a partial-credit rubric for free-form trajectories and audit it against human gold.
- Use Inspect's agent harness to wire a custom tool-use eval end-to-end.

## Lecture chapters

1. [`01-why-agent-eval-breaks-single-turn-scoring.md`](01-why-agent-eval-breaks-single-turn-scoring.md) — why the trajectory replaces the single output as the unit of measurement, the four objects (model adapter, task definition, request type, scorer) re-cast for agent evals, and where this module sits relative to mod-104, mod-107, mod-109, and mod-110.
2. [`02-trajectory-scoring-and-budget-tracking.md`](02-trajectory-scoring-and-budget-tracking.md) — the four signals a trajectory scorer must emit (final-answer correctness, per-step tool-call correctness, budget consumption, termination reason), aggregation over trajectories with bootstrap CIs and cost-vs-accuracy Pareto plots, and the `pass^k` consistency metric.
3. [`03-swe-bench-reproduction-end-to-end.md`](03-swe-bench-reproduction-end-to-end.md) — the SWE-bench / Lite / Verified splits, the five-stage reproduction pipeline (load, generate, build images, run grader, aggregate), the four sources of reproduction gap (harness, prompt, sampling, dataset drift), and the reproducibility manifest that pins `(model, harness, prompt, budget, dataset)`.
4. [`04-interactive-sandboxes-webarena-gaia-agentbench.md`](04-interactive-sandboxes-webarena-gaia-agentbench.md) — sandbox isolation for stateful environments (application state, time, randomness, network egress), the four deterministic-replay strategies (contain, snapshot-replay, frozen-source, accept-and-report), the success-criterion function that grades environment state rather than text, and where AgentBench's portfolio pattern fits.
5. [`05-partial-credit-rubrics-for-trajectories.md`](05-partial-credit-rubrics-for-trajectories.md) — three rubric shapes (milestone checklist, multi-dimensional anchored rubric, free-form judge critique), the mod-105 discipline transferred to trajectory judges (bias controls, anchor design, human calibration), and the audit protocol that produces a `use / re-anchor / do-not-use` decision per dimension.
6. [`06-inspect-agent-harness-end-to-end.md`](06-inspect-agent-harness-end-to-end.md) — a full end-to-end custom agent eval in Inspect: dataset, per-sample seeding solver, `basic_agent` with custom tools, a trajectory scorer that emits the Chapter 2 vector as `Score.metadata`, run and reporting via `inspect eval` / `inspect view`, and re-scoring without re-inferring via `inspect score`.

## Exercises

Five hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after the chapters it depends on.

- [`exercise-01-trajectory-scorer-with-budget-tracking.md`](exercises/exercise-01-trajectory-scorer-with-budget-tracking.md) — build the trajectory-scorer library from Chapter 2 (schema, per-step signals, budget rollup, termination reasons, aggregator with bootstrap CIs and Pareto plot) and demonstrate it against seven seeded synthetic trajectories including a redundant loop, a malformed tool call, a budget-exhausted run, and a harness error.
- [`exercise-02-swe-bench-reproduction.md`](exercises/exercise-02-swe-bench-reproduction.md) — run SWE-bench Verified (or Lite) against a candidate model with a documented harness, produce `% Resolved` and `% Applied`, reproduce a published anchor number within tolerance, and defend any gap against the four Chapter 3 sources.
- [`exercise-03-webarena-or-gaia-run.md`](exercises/exercise-03-webarena-or-gaia-run.md) — stand up either WebArena's contained sandbox or GAIA's open-web environment, run a 30–60 task subset, and write the explicit determinism-strategy argument (probes + gaps) from Chapter 4.
- [`exercise-04-partial-credit-rubric-audit.md`](exercises/exercise-04-partial-credit-rubric-audit.md) — design a two-shape rubric, hand-label a 40-trajectory gold set, run an LLM judge, and produce the per-dimension `use / re-anchor / do-not-use` decision report from Chapter 5.
- [`exercise-05-inspect-agent-eval-end-to-end.md`](exercises/exercise-05-inspect-agent-eval-end-to-end.md) — wire a custom Inspect agent eval end-to-end (dataset, tools, agent solver, trajectory scorer emitting the Chapter 2 vector), run against two models, and demonstrate re-scoring without re-inferring.

Reference solutions live in the paired [`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for primary references — the SWE-bench and SWE-bench Verified papers (Jimenez et al. 2024; OpenAI/Princeton 2024), WebArena (Zhou et al. 2024), GAIA (Mialon et al. 2024), AgentBench (Liu et al. 2024), the Codex `pass@k` paper (Chen et al. 2021) that anchors the estimator, the ReAct pattern paper (Yao et al. 2023), Inspect's official documentation, and the SWE-agent / OpenHands / Aider harness references.
