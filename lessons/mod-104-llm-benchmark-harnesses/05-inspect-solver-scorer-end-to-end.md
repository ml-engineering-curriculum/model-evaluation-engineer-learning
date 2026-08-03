# Inspect: Solver / Scorer Plumbing and Custom Datasets End-to-End

The UK AI Safety Institute's [Inspect](https://inspect.aisi.org.uk/) is the harness UK AISI built to run its pre-deployment safety evaluations against frontier models. It is the most ergonomically opinionated of the four harnesses in this module, and the opinion is: **an eval is a graph of solvers and a scorer**, both first-class, both independently testable, both able to touch tools and sandboxes. lm-eval optimizes for reproducing academic benchmarks; HELM optimizes for cross-model matrices; OpenAI evals optimizes for rubric-graded generation; Inspect optimizes for **multi-turn, tool-using, possibly-sandboxed** evals — the shape most agent, cyber-capability, and jailbreak-resistance evals actually need. This chapter walks Inspect's core abstractions and builds a full custom eval end-to-end.

## The four abstractions

Inspect's model of an eval:

- **`Dataset`** — a list of `Sample` objects. Each `Sample` has an `input` (a string or a list of `ChatMessage`), an optional `target` (the expected answer), an optional `choices` list (for MC-style tasks), and free-form `metadata` for anything else (tags, difficulty, category, source).
- **`Solver`** — a function `Solver: State -> State` that transforms an eval state. Solvers are composable: you chain `prompt_template()`, `system_message()`, `generate()`, `chain_of_thought()`, tool-using solvers, and custom Python solvers into a pipeline. The state is a rich object holding the sample, the message history, the model output, and any intermediate scratch data.
- **`Scorer`** — a function that consumes the final state and produces a `Score` (a value plus explanation and metadata). Scorers include `match()` (regex/exact/word-level match), `includes()` (target as substring), `choice()` (extract a letter for MC tasks), `answer()` (extract from a specific delimiter), `model_graded_fact()` and `model_graded_qa()` (LLM-as-judge), and custom scorers.
- **`Task`** — the binding: a `Task` combines a `Dataset`, a `Solver` (or list of solvers), a `Scorer` (or list), and a config (`epochs`, `sandbox`, `token_limit`, `time_limit`).

The CLI is `inspect eval <task_file>@<task_name> --model <provider/model>`. The framework handles concurrency, retries, error containment, resume-from-crash (via an eval log format), token/time budgeting, and — importantly — sandbox lifecycle.

## Why Inspect is different: solvers are functions of state

lm-eval treats a task as a per-item recipe: "render prompt, issue request, score response." Inspect treats a task as a program on state: "here is a state; each solver can inspect, mutate, and continue it." The consequence is that multi-turn interaction, tool use, and adaptive behavior fall out naturally.

A trivial single-turn eval in Inspect:

```python
from inspect_ai import Task, task
from inspect_ai.dataset import json_dataset
from inspect_ai.solver import generate, system_message
from inspect_ai.scorer import match

@task
def customer_sentiment():
    return Task(
        dataset=json_dataset("data/customer_sentiment.jsonl"),
        solver=[
            system_message("Classify each email as positive, neutral, or negative."),
            generate(),
        ],
        scorer=match(location="any", ignore_case=True),
    )
```

A multi-turn tool-using eval reuses the same shape:

```python
from inspect_ai.solver import use_tools, generate, basic_agent
from inspect_ai.tool import bash, python

@task
def code_repair():
    return Task(
        dataset=json_dataset("data/code_repair.jsonl"),
        solver=basic_agent(
            init=system_message("You are an engineer. Diagnose and fix the failing test."),
            tools=[bash(), python()],
            max_messages=20,
        ),
        scorer=includes(),
        sandbox="docker",
    )
```

The dataset schema and the scorer are unchanged; the solver went from `generate()` to `basic_agent(tools=[bash(), python()])`, and Inspect provisioned a Docker sandbox with those tools available per sample. This orthogonality is the reason Inspect scales to agent evals in a way lm-eval structurally cannot.

## The custom-dataset path

`inspect_ai.dataset` ships three loaders: `csv_dataset`, `json_dataset`, and `hf_dataset`. A JSONL row for the `customer_sentiment` task looks like:

```json
{"input": "The product arrived broken and support has not replied for a week.", "target": "negative", "metadata": {"source_id": "e_00042", "region": "EU"}}
```

The `input` becomes `sample.input`; the `target` becomes `sample.target`; anything else lands in `sample.metadata`. If your source data does not already have this shape, pass a `sample_fields` callable:

```python
from inspect_ai.dataset import FieldSpec, hf_dataset

dataset = hf_dataset(
    path="myorg/customer-sentiment",
    split="test",
    sample_fields=FieldSpec(
        input="email",
        target="label",
        metadata=["region", "source_id"],
    ),
)
```

For anything more complex — mapping into `ChatMessage` lists, filtering, sampling — pass a Python function `(record: dict) -> Sample`.

Two dataset-side patterns worth naming:

- **Chat-style inputs.** If the eval needs a specific system prompt or a partially-completed multi-turn history, produce `input=[ChatMessageSystem(content=...), ChatMessageUser(content=...), ...]` per sample. Solvers will continue the conversation from where the sample left off.
- **Per-sample sandbox config.** Some evals need per-sample sandbox setup (e.g. a specific starting filesystem state for a code-repair task). Attach the setup to `sample.metadata` and let a custom solver run it before `generate()`.

## Solvers in detail

The built-in solvers you will use most:

- **`system_message(content)`** — prepend a system message to the conversation.
- **`prompt_template(template)`** — apply a template to the user input (supports `{prompt}`, `{choices}`, and any `metadata` keys).
- **`generate(cache=False)`** — issue a generation with the current message history, append the model's response, return the updated state.
- **`chain_of_thought(template=...)`** — inject a CoT-style instruction before `generate()`.
- **`multiple_choice()`** — render an MC prompt with lettered choices, generate, and let a downstream scorer extract the letter.
- **`self_critique(model=...)`** — a two-turn solver that generates, then asks the model (or a different model) to critique and revise.
- **`basic_agent(init, tools, max_messages)`** — a ReAct-style tool-using loop that runs until the model produces a terminating message or a message limit is hit. This is the standard on-ramp to agent evals.
- **`use_tools(tools)`** — register tools with the state without starting a loop; useful when combining with custom solvers.

Custom solvers are Python callables decorated with `@solver` that take and return a `TaskState`. You can freely inspect and mutate the message history, call the model via `state.model.generate(...)`, and pass control to the next solver.

```python
from inspect_ai.solver import Solver, TaskState, Generate, solver

@solver
def retry_on_refusal(max_retries=2):
    async def solve(state: TaskState, generate: Generate) -> TaskState:
        for _ in range(max_retries):
            state = await generate(state)
            if not is_refusal(state.output.completion):
                return state
            state.messages.append(ChatMessageUser(
                content="Please answer directly rather than refusing."
            ))
        return state
    return solve
```

Solvers being first-class Python functions is what makes complex evals cleanly modular. In lm-eval the same behavior would require modifying task YAML or subclassing internal classes.

## Scorers in detail

Built-in scorers cover the common cases:

- **`match(location="any", ignore_case=True, ignore_punctuation=True)`** — did the model's output contain the target string somewhere? Configurable normalization.
- **`includes()`** — target as substring, no normalization.
- **`answer(pattern)`** — extract with a regex, then compare.
- **`choice()`** — for `multiple_choice()`-produced states, extract the letter and grade.
- **`model_graded_fact(...)`** — LLM-as-judge for a factual question. Configurable rubric prompt, configurable grader model, returns a categorical `C` (correct) / `I` (incorrect) / `P` (partial) with an explanation.
- **`model_graded_qa(...)`** — LLM-as-judge for open-ended QA with a numerical rubric.

Custom scorers are `@scorer`-decorated callables that receive the final state and the target, and return a `Score`:

```python
from inspect_ai.scorer import scorer, Score, mean, accuracy, Target
from inspect_ai.solver import TaskState

@scorer(metrics=[accuracy(), mean()])
def sentiment_scorer():
    async def score(state: TaskState, target: Target) -> Score:
        response = state.output.completion.lower().strip()
        for label in ["positive", "neutral", "negative"]:
            if label in response:
                return Score(
                    value=1.0 if label == target.text.lower() else 0.0,
                    answer=label,
                    explanation=f"Extracted '{label}' from response.",
                )
        return Score(value=0.0, answer=None, explanation="No label extracted.")
    return score
```

Scorers can be chained. `scorer=[choice(), model_graded_qa(model="openai/gpt-4o-mini")]` runs both and reports both. Aggregation metrics (`accuracy()`, `mean()`, `stderr()`, `bootstrap_stderr()`) are declared at the scorer decorator level.

## Sandboxed tool use

Inspect's sandbox integration is what makes it viable for cyber and agent evals where the model executes untrusted code. Sandboxes are provisioned per-sample, configured via a `sandbox` argument to `Task` (or per-sample via metadata). The two most common:

- **`sandbox="docker"`** with a `compose.yaml` in the eval directory. Each sample gets a fresh container. Tools like `bash()` and `python()` run inside it.
- **`sandbox="local"`** — no isolation; runs in the host process. Fine for prototyping, not for anything that runs untrusted model output.

For a code-repair task, a `compose.yaml` alongside the task file:

```yaml
services:
  default:
    build: .
    working_dir: /app
    command: sleep infinity
```

The `Dockerfile` provides the base environment. Inspect handles container lifecycle: bring up per sample, run the solver's tool calls, bring down. Failures are captured in the eval log per-sample, not fatal to the run.

## Running the eval

```bash
inspect eval customer_sentiment.py@customer_sentiment \
  --model openai/gpt-4o-mini \
  --limit 500 \
  --epochs 3 \
  --log-dir logs/customer_sentiment
```

`--epochs N` re-runs each sample N times, which is how Inspect handles the variance of sampled generation without you writing a loop. Results are aggregated across epochs. The eval log format under `--log-dir` captures every sample's full transcript (all messages, tool calls, tool responses, model output) plus the scorer verdict and per-metric aggregates. `inspect view` opens an interactive log browser.

For multi-model runs, `--model` can be repeated. For batch parallelism, `--max-connections` throttles concurrent requests to each provider.

## Cross-harness comparison

Same customer-sentiment task, four harnesses. The Inspect version:

- **Dataset** — `hf_dataset(...)` with a `sample_fields=FieldSpec(input="email", target="label")`.
- **Solver** — `[system_message("Classify..."), generate()]`.
- **Scorer** — `match(location="any", ignore_case=True)` or a custom `sentiment_scorer()`.

Compared to Chapter 2's lm-eval YAML, the Inspect version is more Python and less config; the payoff is that swapping in a `basic_agent(tools=[...])` solver to test tool-using variants is a one-line change, whereas lm-eval would require a new task type entirely. For pure classification the lm-eval YAML is more compact. For anything with tools or multi-turn state, Inspect is dramatically shorter and clearer.

## The eval log format and reproducibility

Every Inspect run produces an eval log — a self-contained JSON-with-metadata file that records the resolved task config, model, decoding params, per-sample transcripts, per-sample scorer verdicts, aggregate metrics, and the full stack of solver / scorer names and their arguments. The log is the reproducibility artifact. Two consequences:

- **Resume after crash.** `inspect eval-retry <log_file>` re-runs only the samples that errored or were not yet run.
- **Re-score without re-inferring.** `inspect score <log_file> --scorer new_scorer.py@new_scorer` runs a different scorer over the same transcripts. This is how you cheaply try three grader rubrics without re-paying for inference. It is one of Inspect's most under-appreciated features.

For reproducibility, the log is what you share with a reviewer. `inspect view logs/customer_sentiment` opens the interactive viewer; the reviewer can click into any sample, see the full transcript, see the scorer's explanation, and audit the eval end-to-end. The lm-eval `--log_samples` JSONL is comparable but less structured; Inspect's log is the richest reproducibility artifact of the four harnesses.

## When Inspect is the right (and wrong) tool

**Right for.** Multi-turn evals, agent evals with tools, jailbreak-resistance evals with adaptive attackers, sandboxed cyber-capability evals, evals where you need to re-score existing transcripts without re-inferring, evals where the scorer itself is complex (LLM-as-judge with a custom rubric, procedural scorers that inspect intermediate state).

**Wrong for.** Pure log-likelihood ranking against a base model. Inspect can do it but the ergonomics are optimized for chat/generation; lm-eval is a better fit for that shape. Also wrong when you specifically need to reproduce an lm-eval or HELM published number — you should use the harness that produced the number, not a reimplementation, because the reimplementation gaps will dominate the actual signal.

## Summary

Inspect's four abstractions — `Dataset`, `Solver`, `Scorer`, `Task` — treat an eval as a program on state rather than a per-item recipe, which is what lets it scale cleanly to multi-turn, tool-using, sandboxed evaluations. A `Dataset` produces `Sample` objects; a `Solver` chain transforms `TaskState`; a `Scorer` verdicts the final state; `Task` binds them and adds config. Custom datasets, custom solvers, and custom scorers are Python functions and are the right modularity choice for evals with meaningful control flow. Sandboxed tool use is first-class via Docker Compose. The eval log format is the richest reproducibility artifact of any harness in this module; re-scoring without re-inferring is uniquely cheap. Inspect is the right harness when the eval has multi-turn structure or tool use; it is not the right harness when the goal is to reproduce a public lm-eval or HELM number. Chapter 6 turns to the OpenAI evals registry, which specializes at the opposite end of the spectrum: single-turn, rubric-graded, model-as-judge scoring.
