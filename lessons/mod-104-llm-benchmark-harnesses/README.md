# mod-104-llm-benchmark-harnesses: LLM Benchmark Harnesses: lm-eval-harness, HELM, OpenAI Evals, Inspect

**Estimated effort:** 16 hours

The previous three modules taught you how to define an eval, source and version its data, and report classical metrics defensibly. This module is where that discipline meets the tooling the field actually uses to run LLM benchmarks: EleutherAI's [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness), Stanford CRFM's [HELM](https://crfm.stanford.edu/helm/), OpenAI's [evals](https://github.com/openai/evals) registry, and UK AISI's [Inspect](https://inspect.aisi.org.uk/). Every published LLM leaderboard number, every internal launch report, and every "we ran it on our copy of MMLU" claim goes through one of these harnesses. If you cannot register a task, run it, and reproduce a public number in one of them, you cannot participate in the shared measurement infrastructure the field uses.

The chapters are deliberately tool-specific. Each of the four harnesses makes different assumptions about *what an eval is* — whether items are scored by log-likelihood ranking, by generated-string equality, by a rubric-scored judge model, or by a multi-turn solver with tools — and those assumptions leak into every reproducibility incident. The last two chapters return to the shared pathology: prompt-format sensitivity and the reproducibility protocol that keeps the numbers meaningful across a model swap, a harness upgrade, or a dataset rev.

## Learning objectives

- Register and run a new task in EleutherAI lm-evaluation-harness end-to-end with both log-likelihood and generation evaluators.
- Reproduce a public lm-eval-harness or HELM result within reported tolerance and explain any gap (prompt format, decoding, normalisation).
- Build an Inspect (UK AISI) eval with solver/scorer plumbing and a custom dataset.
- Build an OpenAI evals-style registry entry with model-graded scoring.
- Diagnose prompt-format sensitivity, log-prob-vs-generation gap, and per-task reproducibility (seeds, decoding params, dataset hash).

## Lecture chapters

1. [`01-what-a-benchmark-harness-is.md`](01-what-a-benchmark-harness-is.md) — why the field runs evals through shared harnesses, the four objects every harness carries (model adapter, task definition, request batcher, scorer), and the reproducibility contract each one implicitly promises.
2. [`02-lm-eval-harness-task-registration.md`](02-lm-eval-harness-task-registration.md) — EleutherAI lm-evaluation-harness anatomy, the YAML `task` definition, `doc_to_text` / `doc_to_target` / `doc_to_choice`, few-shot construction, and running against a local or served model.
3. [`03-loglikelihood-vs-generation-evaluators.md`](03-loglikelihood-vs-generation-evaluators.md) — the `loglikelihood`, `loglikelihood_rolling`, and `generate_until` request types; when multiple-choice-by-ranking disagrees with multiple-choice-by-generation; length normalisation, byte normalisation, and how the two evaluators diverge on the same task.
4. [`04-helm-scenarios-and-reproducing-a-public-number.md`](04-helm-scenarios-and-reproducing-a-public-number.md) — HELM's `Scenario` / `Adapter` / `Metric` / `RunSpec` decomposition, how to run a scenario, what "reproducing within tolerance" means in practice, and the four gap sources you actually see (prompt format, decoding, normalisation, dataset rev).
5. [`05-inspect-solver-scorer-end-to-end.md`](05-inspect-solver-scorer-end-to-end.md) — the Inspect `Task` / `Dataset` / `Solver` / `Scorer` model, custom datasets, tool-using solvers, sandboxed execution, and where Inspect fits versus lm-eval-harness for safety and agent-style evals.
6. [`06-openai-evals-registry-and-model-graded.md`](06-openai-evals-registry-and-model-graded.md) — the OpenAI evals registry format, `Eval` and `CompletionFn` abstractions, `modelgraded` YAML specs, rubric prompt design, and how a model-graded eval is validated against a human-labelled slice.
7. [`07-prompt-format-sensitivity-and-reproducibility.md`](07-prompt-format-sensitivity-and-reproducibility.md) — chat template vs. completion prompt, tokenizer edge cases, decoding parameter and seed hygiene, dataset-hash and harness-version pinning, and the per-task reproducibility protocol that ties the four preceding chapters together.

## Exercises

Five hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after finishing the chapters it depends on.

- [`exercise-01-lm-eval-harness-custom-task.md`](exercises/exercise-01-lm-eval-harness-custom-task.md) — register a new task in lm-evaluation-harness with both a log-likelihood and a generation evaluator, run it against a local model, and report per-item disagreement between the two modes.
- [`exercise-02-helm-scenario-reproduction.md`](exercises/exercise-02-helm-scenario-reproduction.md) — reproduce a published HELM or lm-eval-harness result within tolerance, write a gap-analysis report, and pin the exact configuration that reproduces the number.
- [`exercise-03-inspect-eval-end-to-end.md`](exercises/exercise-03-inspect-eval-end-to-end.md) — build an Inspect eval end-to-end (dataset loader, solver, scorer, task) against a custom dataset, then run it against two models and produce a signed run bundle.
- [`exercise-04-openai-evals-registry-task.md`](exercises/exercise-04-openai-evals-registry-task.md) — write an OpenAI-evals-style registry entry with a model-graded rubric, validate the grader against a human-labelled subset, and report grader-vs-human agreement.
- [`exercise-05-prompt-format-sensitivity-diagnostic.md`](exercises/exercise-05-prompt-format-sensitivity-diagnostic.md) — sweep prompt format, decoding, and normalisation on a single task and produce a sensitivity report that separates task signal from harness-plumbing noise.

Reference solutions live in the paired [`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for primary references — the lm-evaluation-harness paper (Gao et al. 2023) and repository, the HELM paper (Liang et al. 2022) and Stanford CRFM documentation, the Inspect framework docs from UK AISI, the OpenAI evals repository and registry spec, the prompt-sensitivity literature (Sclar et al. 2024, Alzahrani et al. 2024, Mizrahi et al. 2024), and the reproducibility work behind the Open LLM Leaderboard v2 replication.
