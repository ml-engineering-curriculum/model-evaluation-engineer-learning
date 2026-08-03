# What a Benchmark Harness Is (And Why the Field Uses Four of Them)

Every published LLM benchmark result is the output of a *harness* — a piece of software that owns the loop from "here is a dataset item" to "here is a scored number." The reason the field has converged on shared harnesses instead of one-off scripts is not aesthetics. It is because the same task, evaluated by two teams with two different scripts, will typically produce different numbers, and the discipline of pointing at the same harness, the same task version, and the same dataset hash is the only cheap way to make results comparable. This chapter is about what a harness actually does, the four objects that every harness carries (in different clothes), and why the four systems this module covers — EleutherAI lm-evaluation-harness, HELM, OpenAI evals, and Inspect — each specialize toward a different failure mode of that shared measurement problem.

## What "harness" means

A benchmark harness is the code that, given a model and a task definition, produces a scored eval run. Concretely it does five things:

1. **Load the task.** Read the dataset (from HF Hub, a local directory, a URL, a scenario spec), materialize items in a schema the harness understands, and produce the split you asked for.
2. **Render each item into a prompt.** Apply few-shot examples, system prompts, chat templates, and any task-specific transforms so the model sees the exact text the eval intends.
3. **Issue requests against the model adapter.** Batch and route requests to the model backend — a local Hugging Face model, a llama.cpp binary, a vLLM server, an OpenAI-compatible endpoint. The harness does not care which; the *model adapter* hides that.
4. **Collect responses.** Log-likelihoods over candidate continuations, generations up to a stop condition, or tool-mediated multi-turn interactions.
5. **Score and aggregate.** Apply per-item scorers, apply length or byte normalisation, aggregate to a benchmark-level number with its uncertainty, and emit a run artifact.

Any script you write to "evaluate a model on X" is doing these five things. What makes a *harness* different from a script is that these five stages are separately configurable, versioned, and reusable across tasks and model backends. That separation is what makes a benchmark result cite-able rather than one-off.

## The four objects every harness carries

The four systems this module covers use different vocabulary, but every one of them has an analogue for these four objects. Recognizing them explicitly makes it much easier to move between harnesses.

**1. Model adapter.** A thin interface over "here is a prompt, give me back a log-likelihood or a generation." lm-evaluation-harness calls this an `LM` subclass (`HFLM`, `vLLM`, `OpenAICompletionsAPI`, `Anthropic`, etc.). HELM calls it a `Client`. Inspect calls it a `Model`. OpenAI evals calls it a `CompletionFn`. The important properties of a model adapter are (a) whether it can return log-likelihoods over provided continuations, not just generate them, (b) what maximum context it supports, and (c) whether it applies its own chat template or expects a fully-rendered string.

**2. Task definition.** A specification of the dataset, the prompt rendering, the expected output format, and which scorers apply. In lm-evaluation-harness this is a YAML file with `dataset_path`, `doc_to_text`, `doc_to_target`, `output_type`, and `metric_list`. In HELM it is a `Scenario` (dataset producer) plus a `RunSpec` that binds the scenario to an adapter and metrics. In Inspect it is a Python `Task` that returns a `Dataset`, a `Solver` chain, and a `Scorer`. In OpenAI evals it is a YAML registry entry plus a JSONL sample file and, for graded evals, a `modelgraded` spec.

**3. Request type / decoding contract.** The two dominant request shapes are *log-likelihood* (score one or more provided continuations under the model, no sampling) and *generation* (produce tokens until a stop condition, with a decoding config). A third — *tool-mediated multi-turn* — is what Inspect (and, increasingly, HELM) support for agent-style evals. The request type is the single biggest source of "same model, same task, different number" gaps, and Chapter 3 is entirely about the log-likelihood vs. generation split.

**4. Scorer and aggregator.** A per-item function `(prediction, target) -> {"metric": value, ...}` plus a set of aggregators (mean, macro-F1, pass@k, model-graded rubric, exact-match with normalisation). The scorer is where the *definition* of the metric lives. "Exact match" in one harness will strip punctuation, lowercase, and canonicalize whitespace; in another it will not. If your reproduction is off by a few points, the scorer is where the discrepancy is often hiding.

Every harness is a specific ergonomic choice about how those four objects are wired together and what defaults they carry. Once you can point at each object in an unfamiliar harness, learning the next harness is largely a matter of translating vocabulary.

## Why four harnesses, not one

The systems this module covers are not redundant. Each one grew from a different measurement problem and pays for that specialization with rougher ergonomics in the others' domain.

**EleutherAI lm-evaluation-harness (lm-eval).** Grew out of GPT-Neo/GPT-J evaluation and the need for a single loop that can run hundreds of academic tasks against a local checkpoint. Optimizes for: many-task batch evaluation, log-likelihood evaluators, easy addition of new tasks via YAML, and precise reproduction of the "canonical" number for tasks like MMLU, HellaSwag, ARC, TruthfulQA. It is the harness the Hugging Face Open LLM Leaderboard uses under the hood. If a paper says "evaluated on the standard suite," it usually means through lm-eval. Weaker for open-ended generation, judge models, and multi-turn agent evals.

**HELM (Holistic Evaluation of Language Models).** Grew out of Stanford CRFM's argument that no single number captures a language model and that a serious eval report is a matrix over many scenarios and many metrics reported *together*. Optimizes for: broad, standardized cross-model comparison; a stable `Scenario`/`Adapter`/`Metric` decomposition; and a hosted leaderboard with full run traces. Where lm-eval is best at "reproduce this one number," HELM is best at "compare N models on M scenarios and read the whole matrix." The trade-off is heavier configuration and slower iteration on a single novel task.

**OpenAI evals.** Grew out of OpenAI's internal need to grade generations from chat-tuned models at scale, where log-likelihood ranking often does not apply (a chat model may refuse to be scored token-by-token) and rubric-graded scoring is the pragmatic answer. Optimizes for: registry-based task definitions in YAML, JSONL sample files, and — the distinguishing feature — first-class support for **model-graded evaluation** with configurable rubrics. If your eval question is "did the model do a good job at this open-ended task, in the judgment of another capable model," this harness is where the community-shared rubric templates live.

**Inspect (UK AI Safety Institute).** Grew out of pre-deployment safety evaluation, where the eval needs to be a possibly-long, tool-using, multi-turn interaction that runs inside a sandbox (a container, a filesystem, a shell), with a scorer that inspects both final answers and intermediate trajectories. Optimizes for: agent-style evals, dangerous-capability evals (cyber, biosecurity, autonomy), sandboxed tool use, and cleanly separable `Solver` and `Scorer` plumbing. It is what UK AISI itself uses for its published pre-deployment model evaluations.

The rest of this module goes deep on each. The takeaway from this chapter is that the harness you pick is a strong prior on what kind of eval you are running, and running the wrong task on the wrong harness — a long multi-turn agent task on lm-eval, a log-likelihood MMLU reproduction on Inspect — is a common source of avoidable friction.

## The reproducibility contract

The reason harnesses exist is reproducibility, so it is worth being explicit about what they promise and what they do not.

**What a harness promises.** Given the same task version, the same dataset hash, the same model checkpoint, the same decoding config, and the same random seed, a well-behaved harness will produce the same score. That is the contract. It is what lets one team point at a leaderboard number and another team verify it. Every harness in this module lets you dump the exact request/response pairs it issued, precisely so someone else can rerun them.

**What a harness does not promise.** Different harnesses running the "same" task on the same model will typically *not* produce the same number, and this is not a bug. lm-eval's `hellaswag` and HELM's `hellaswag` differ in prompt template, few-shot selection, and log-likelihood normalisation — each choice defensible, but the numbers are not directly comparable. This is why the reproducibility protocol (Chapter 7) always pins the *harness plus version*, not just the task name.

**What breaks reproducibility across harness versions.** The single biggest one is prompt-template changes when the harness silently adjusts a chat model's system prompt or renders `bos_token` differently. The lm-eval MMLU scores that ship with different harness releases can move by 1–3 points on the same model just from prompt-template tweaks. This is why upstream harness releases usually publish a "here is what changed and here is the numeric impact per benchmark" note, and why your project should pin the version aggressively.

## A concrete comparison

To make the four-objects taxonomy concrete, here is the same conceptual task — "given a passage and a question with four choices, pick the correct choice" — written as it looks in each of the four harnesses. Details of each are covered in the subsequent chapters; the point here is to see the shape.

**lm-eval (YAML).** The task ships as a YAML file with `dataset_path`, `output_type: multiple_choice`, `doc_to_text`, `doc_to_choice`, and a metric declaration. The harness issues `loglikelihood` requests against each choice and picks the highest-scoring one.

```yaml
task: my_mc_task
dataset_path: hf-org/my-mc-dataset
output_type: multiple_choice
doc_to_text: "Passage: {{passage}}\nQuestion: {{question}}\nAnswer:"
doc_to_choice: "{{[choice_a, choice_b, choice_c, choice_d]}}"
doc_to_target: "{{answer_idx}}"
metric_list:
  - metric: acc
    aggregation: mean
```

**HELM.** The scenario is a Python class producing `Instance` objects; a `RunSpec` binds the scenario to the multiple-choice adapter and the `ExactMatch` metric.

```python
run_spec = get_run_spec(
    scenario_spec=ScenarioSpec(class_name="my_mc.MyMCScenario"),
    adapter_spec=get_multiple_choice_joint_adapter_spec(),
    metric_specs=get_exact_match_metric_specs(),
)
```

**Inspect (Python).** A `Task` returns a dataset of `Sample` objects, a solver chain (typically `generate()` for a straight completion), and a scorer such as `choice()` that extracts and grades a letter response.

```python
@task
def my_mc_task():
    return Task(
        dataset=json_dataset("data/my_mc.jsonl"),
        solver=[multiple_choice()],
        scorer=choice(),
    )
```

**OpenAI evals.** A YAML registry entry names the eval class, points at a JSONL samples file, and declares how the completion is scored. For multiple choice, `evals.elsuite.basic.match:Match` compares the generation to an expected string.

```yaml
my_mc_task:
  id: my_mc_task.dev.v0
  description: My MC task
  metrics: [accuracy]
my_mc_task.dev.v0:
  class: evals.elsuite.basic.match:Match
  args:
    samples_jsonl: my_mc/samples.jsonl
```

Same task, four shapes. Once you can look at any one of them and identify the model adapter, task definition, request type, and scorer, the rest is filling in the harness-specific vocabulary.

## Summary

A benchmark harness is the software that turns a task definition and a model adapter into a scored eval run. Every harness carries four objects — model adapter, task definition, request type / decoding contract, and scorer — in different clothes. The four systems this module covers each specialize toward a different measurement problem: lm-eval-harness for reproducible log-likelihood academic benchmarks, HELM for holistic multi-scenario comparison, OpenAI evals for model-graded open-ended generation, Inspect for sandboxed multi-turn safety and agent evals. A harness's reproducibility contract is: same task version, dataset hash, model checkpoint, decoding config, and seed produce the same score. What it does *not* promise is that two harnesses' "same" task give the same number — the differences in prompt template, normalisation, and scorer are exactly why the field pins harness plus version, not just the task name. Chapter 2 opens the first of the four harnesses in detail.
