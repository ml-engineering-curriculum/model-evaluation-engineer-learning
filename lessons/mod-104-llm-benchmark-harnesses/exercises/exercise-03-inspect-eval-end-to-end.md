# exercise-03: Build an Inspect Eval End-to-End

**Estimated effort:** 4 hours

## Objective

Build a working [Inspect](https://inspect.aisi.org.uk/) eval end-to-end against a custom dataset: dataset loader → solver → scorer → task binding → run → signed run bundle. The eval must use at least one non-trivial solver behavior (a multi-turn solver, a tool-using solver, or a chained solver with a custom Python step) and a scorer that is more than a trivial exact-match. Run the eval against **two** models, produce the eval logs, and ship a reproducibility bundle. The point is to feel Inspect's `Solver`/`Scorer` composition working for a real task and to see the eval-log reproducibility story that Chapter 5 promised.

## Prerequisites

- mod-104 Chapter 5 (Inspect).
- `inspect-ai` installed (`pip install inspect-ai`) — see the [Inspect docs](https://inspect.aisi.org.uk/) for setup.
- API access to at least two models. Two cheap options: OpenAI `gpt-4o-mini` + Anthropic `claude-haiku`, or two open-weights models served via `vllm` behind an OpenAI-compatible endpoint (Inspect supports `openai/<model>` and provider-specific configs).
- If your task uses tool sandboxing, Docker installed and working.

## The task: pick one

Pick a task shape that requires more than a single-turn generation-and-string-match. Suggested shapes and dataset seeds:

- **Multi-turn QA with a scratchpad solver.** Build a chain-of-thought + self-critique solver: `[system_message("..."), generate(), self_critique(), generate()]`. Dataset seed: a subset of `hendrycks/competition_math`, `openbookqa`, or `strategyqa`. Scorer: `match()` against a numeric answer *or* `model_graded_fact()`.
- **Tool-using code repair.** Given a broken Python function and a failing pytest, the model must edit the code and confirm the test passes. Solver: `basic_agent(tools=[bash(), python()], max_messages=20)`. Sandbox: `docker`. Dataset seed: your own 20-item mini-benchmark of trivial bugs in short Python functions, or a subset of `princeton-nlp/SWE-bench` (small variant).
- **Multi-hop factual QA with a browsing tool.** If you have a search-tool integration handy, `basic_agent(tools=[web_search(), web_browse()])`. Otherwise use a mocked/scripted tool that returns preloaded documents. Dataset seed: `hotpot_qa`.
- **Jailbreak-resistance eval.** Given a safe-content prompt + a paraphrased jailbreak attempt, does the model comply or refuse? Solver: `[generate()]`. Scorer: `model_graded_qa(...)` with a refusal rubric. Dataset seed: subset of `walledai/JailbreakBench` — pick a small, clearly-labelled subset.
- **Custom internal task** you can legitimately share. Anything with meaningful control flow (multi-turn, tool-using, or with a non-trivial scorer) qualifies.

Cap the dataset size to 50–200 items so this runs in an evening. If your task is expensive (`basic_agent` runs 20 messages per sample), 50 items is plenty.

## Requirements

### Part A — dataset

Convert your source data into an Inspect-compatible format. Either:

- Write a JSONL file with one `Sample`-shaped record per line (`input`, `target`, optional `metadata`) and load with `json_dataset(...)`.
- Wrap an HF dataset with `hf_dataset(path=..., split=..., sample_fields=FieldSpec(input=..., target=..., metadata=[...]))`.
- Write a Python function `record_to_sample(record: dict) -> Sample` and pass it to whatever loader is appropriate.

Requirements:

- **Every sample has a stable `id`** (either the source's ID or an assigned one). This is what makes per-item re-scoring possible.
- **Every sample has meaningful `metadata`** (source id, difficulty, category — whatever will let you slice results later).
- **Ship the JSONL (or a script that produces it)** in the deliverables; a reviewer must be able to regenerate it.

### Part B — solver chain (non-trivial)

The solver must be more than a single `generate()` call. At minimum:

- A `system_message(...)` or `prompt_template(...)` step configuring the task instruction.
- The main solver: either
  - a **multi-step chain** (`chain_of_thought`, `self_critique`, or a hand-rolled `@solver` Python function that inspects state and conditionally continues), or
  - a **tool-using agent** (`basic_agent(tools=[...])`), or
  - a **custom `@solver`** that implements a non-trivial control flow specific to your task (retry-on-refusal, adaptive-difficulty follow-ups, evidence-gathering loop).

Document, in the code, what the solver is doing and why the pipeline is shaped that way.

If you use tools, provide a sandbox config (`sandbox="docker"` + `compose.yaml`) alongside the eval file. Non-sandboxed tool use is acceptable only if the tool cannot run arbitrary code.

### Part C — scorer (also non-trivial)

The scorer must be one of:

- A `model_graded_fact()` or `model_graded_qa()` scorer with a **custom rubric prompt** (not the default) and a stated grader model. Rubric design follows the constraints in Chapter 6.
- A **custom `@scorer`** callable that inspects the final state and produces a `Score` with a nontrivial `value` (partial credit, multi-criterion breakdown, or a computed metric like unit-test pass rate).
- A **chain of scorers** (`scorer=[...]`) reporting multiple metrics — e.g. an exact-match floor plus a model-graded verdict on top.

Aggregation metrics (`accuracy()`, `mean()`, `bootstrap_stderr()`) must be declared on the scorer. The final eval log should report a CI or SE on every metric.

### Part D — task binding and configuration

Write a `Task` decorated with `@task`, binding dataset + solver + scorer:

```python
from inspect_ai import Task, task

@task
def mytask(some_param: str = "default"):
    return Task(
        dataset=<dataset>,
        solver=<solver_chain>,
        scorer=<scorer>,
        epochs=3,
        token_limit=8192,
        # sandbox="docker",  # only for tool-using tasks
    )
```

Requirements:

- **`epochs` > 1** if the task involves sampling. Report the aggregation across epochs.
- **`token_limit` or `time_limit`** is set — an unbounded eval is a footgun.
- **The task takes at least one parameter** (e.g. `grader_model`, `max_messages`, `system_prompt_variant`) so you can run controlled variants with `inspect eval mytask.py@mytask --task-arg <name>=<value>`.

### Part E — run against two models

Run the eval against two models with `inspect eval ... --model <provider/model> --log-dir logs/<model_id>`.

- Both runs must be logged under the same `--log-dir` structure so they can be compared side by side.
- Use `inspect view logs/` to open the interactive log viewer and spot-check per-sample transcripts.
- The two models should be different families (e.g. an OpenAI model and an Anthropic model, or an open-weights model and a hosted model). Comparing two OpenAI models is fine as a fallback but less interesting.

### Part F — the report

Write `REPORT.md` (≤ 2 pages) with:

1. **Task description** — one paragraph on what the eval measures, why the solver looks the way it does, and why the scorer is the right one.
2. **Results table** — per model, headline metric with 95% CI (`bootstrap_stderr`), token cost, wall time, and error rate (fraction of samples that errored or hit a limit).
3. **Per-sample sanity check** — 3 samples of your choice with the input, the model's final response, and the scorer's verdict. Include one where the two models disagreed if you can find one.
4. **Solver / scorer behavior notes** — for tool-using or multi-turn evals, one paragraph on how many messages / tool calls per sample on average, and any pathological cases (infinite loops truncated by `max_messages`, tool errors dominating one model's failures, etc.).
5. **Reproducibility handoff** — the pinned versions and the command to rerun.

### Part G — the signed run bundle

Package the deliverables as a zip / tarball:
- `mytask.py` (the eval)
- `data/` (the JSONL or a `build_dataset.py` script + input pointers)
- `compose.yaml` + `Dockerfile` (if applicable)
- `logs/<model_id>/*.eval` (Inspect eval logs — one per model)
- `MANIFEST.md`
- `REPORT.md`
- `run.sh`

Optionally sign the bundle with `sha256sum */* > MANIFEST.sha256`. (Full Sigstore signing is a stretch goal but not required.)

## Starter guidance

- **Start with a trivial `generate()` solver against a 5-item dataset and confirm the plumbing.** Then swap in your intended solver. Debugging Inspect solver chains is much easier with a working baseline to A/B against.
- **Turn on `inspect view` early.** The interactive viewer surfaces solver / scorer errors that the CLI summarizes into a single sentence. Most first-time bugs — bad tool schema, malformed rubric, forgotten `await` in a custom solver — are obvious in the viewer.
- **For model-graded scorers, spot-check the grader on 5 hand-picked samples before running the full eval.** Chapter 6's rubric-design rules apply — constrain the output space, include the reference, prefer few coarse categories.
- **Do not skip `epochs > 1` for sampled runs.** A single-epoch sampled run has meaningless CIs.
- **If you use `basic_agent`, be defensive about `max_messages`.** A model that starts producing tool calls in a loop will burn your quota fast; 20 is a reasonable default for a first cut.
- **For sandboxed evals, budget for image build time.** A first-run `sandbox="docker"` eval spends real minutes on the initial image build.
- **Use `inspect eval-retry` to recover from transient failures** without re-running the successful samples. Real eval runs have provider timeouts and rate limits; the retry ergonomics save cost.
- **The `inspect score` command re-scores an existing log** — if you want to try three grader rubrics, run the eval once and re-score three times. This is uniquely cheap on Inspect and is worth exercising in the exercise.

## Acceptance criteria

- The eval runs end-to-end via a single `inspect eval` invocation captured in `run.sh`.
- The solver is non-trivial (multi-step, tool-using, or a custom `@solver` with meaningful control flow), and its shape is documented.
- The scorer is non-trivial (custom `@scorer`, model-graded with custom rubric, or a chain of scorers), with aggregation metrics declared.
- The eval is run against two distinct models with logs under a shared `--log-dir`.
- `REPORT.md` includes a results table with 95% CIs, three per-sample sanity-check transcripts, and a solver-behavior paragraph.
- The eval logs (`.eval` files) are included in the bundle and viewable with `inspect view`.
- `MANIFEST.md` pins the harness version, model IDs, dataset revision, seeds, and library versions per Chapter 7.
- If tools are used, a working `compose.yaml` is shipped and the sandbox is exercised.
- The bundle is self-contained: a reviewer with the API credentials could run `bash run.sh` and produce comparable numbers.

## Stretch goals

- **Re-score without re-inferring.** After the initial run, write a second scorer (a different rubric, a different metric, or a stricter regex) and use `inspect score logs/<model_id>/... --scorer new_scorer.py@new_scorer` to re-score the same transcripts. Report the two scores side-by-side and comment on which is more informative.
- **Sandbox stress test.** For a tool-using eval, deliberately give the model a sample where the "obvious" tool sequence will fail (e.g. a syntax error in the setup). Observe how the solver handles it and whether the scorer's verdict is correct.
- **Custom metric with subgroup breakdown.** Add a per-slice metric (e.g. per-`metadata.category` accuracy) via a custom scorer aggregation. Report the per-slice table alongside the aggregate.
- **Adversarial solver variant.** Add a second `@task` that uses a different solver chain (e.g. with vs. without `self_critique`) and report the delta. This is a preview of exercise-05's sensitivity work applied to solver shape rather than prompt format.
- **Full Sigstore signing** of the run bundle. `cosign sign-blob logs/<model_id>/eval.eval` and include the signature in the deliverables.
