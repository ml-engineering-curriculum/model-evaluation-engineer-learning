# OpenAI Evals: Registry Format and Model-Graded Scoring

OpenAI's [evals](https://github.com/openai/evals) framework was released in 2023 to support the pattern that dominates real chat-model evaluation: the answer is open-ended text, and the scorer is *another language model* applying a rubric. Log-likelihood ranking does not apply to a response like "Explain photosynthesis at a middle-school level"; string exact-match applies only trivially; but a well-designed rubric prompt run through a capable model can grade it consistently, cheaply, and at scale. This chapter is about the OpenAI evals registry format — the YAML/JSONL surface for declaring tasks and rubrics — and about the discipline of validating a model-graded eval against human labels so the number it reports is worth reading. mod-105 goes deep on LLM-as-judge platforms and their pathologies; this chapter is the harness-side introduction.

## The registry model

OpenAI evals treats an eval as a **registry entry** plus a **samples file**. The registry entry is YAML; the samples file is JSONL. The registry is scanned at CLI time and the eval is looked up by name.

**Registry entry (YAML).** Typically two YAML documents in the same file. The first declares the eval group; the second declares the version:

```yaml
customer_sentiment:
  id: customer_sentiment.dev.v0
  description: Three-way sentiment classification of customer emails.
  disclaimer: Internal task, not comparable to public sentiment benchmarks.
  metrics: [accuracy]

customer_sentiment.dev.v0:
  class: evals.elsuite.basic.match:Match
  args:
    samples_jsonl: customer_sentiment/samples.jsonl
```

The top group `customer_sentiment` names the current version (`customer_sentiment.dev.v0`) and metadata. The `.dev.v0` document binds an `Eval` Python class (`evals.elsuite.basic.match:Match` is the shipped exact-match evaluator) and its arguments — most importantly the samples file.

**Samples file (JSONL).** One JSON object per line. Each row is passed to the `Eval` class as an input. For the shipped `Match` eval, each sample has `input` (a chat-message list) and `ideal` (the correct answer or a list of acceptable answers):

```json
{"input": [{"role": "system", "content": "Classify each email as positive, neutral, or negative."}, {"role": "user", "content": "The product arrived broken and support has not replied for a week."}], "ideal": "negative"}
```

The registry lives under `evals/registry/evals/` (for public / shipped evals) or a user-supplied `--registry_path` (for private evals). Samples files live under `evals/registry/data/<eval_name>/`.

**Running the eval.** The `oaieval` CLI takes a `completion_fn` (a model reference) and the eval group name:

```bash
oaieval gpt-4o-mini customer_sentiment \
  --registry_path ./my_registry \
  --record_path logs/customer_sentiment.jsonl
```

The record path collects the full per-sample event log — prompt, completion, scorer verdict, per-sample metric. The aggregate metric prints to stdout and is included in the record.

## Completion functions: what "the model" is

`CompletionFn` is the OpenAI-evals term for the model adapter. The default `CompletionFn` implementations wrap OpenAI's completion / chat completion APIs, but the framework lets you register your own for any backend: local models via `openai-python`-compatible endpoints (vLLM, TGI, llama.cpp), Anthropic API, community adapters for Google and Cohere, and pipeline `CompletionFn`s that transform inputs or aggregate multiple calls.

Two properties of `CompletionFn` matter for eval design:

- **`CompletionFn` is generation-only in the framework's mainline.** Log-likelihood over provided continuations is not the primary path. If your eval fundamentally needs log-likelihood ranking, OpenAI evals is not the right harness — go back to lm-eval.
- **`CompletionFn` composition.** A `CompletionFn` can wrap another `CompletionFn`. This is how you build tool-using or chain-of-thought variants without touching the eval itself: pass `--completion_args wrap=cot` (or your own wrapper) and the same eval is run against the wrapped model.

## Built-in eval classes

`evals.elsuite` ships several base classes; the ones you will use repeatedly:

- **`basic.match:Match`** — the model's generation is compared to `ideal` with exact-match or substring-match (configurable). Handles the "did the model produce the expected string" family.
- **`basic.fuzzy_match:FuzzyMatch`** — case-insensitive, punctuation-normalized match.
- **`basic.includes:Includes`** — the ideal appears anywhere in the response.
- **`basic.json_match:JsonMatch`** — parse the response as JSON and match against `ideal`.
- **`modelgraded:ModelBasedClassify`** — the flagship model-graded family. See the next section.
- **`cot_classify:CoTClassify`** — chain-of-thought variant that prompts the model to reason before producing the classification, then extracts the final answer.

For most novel evals you will either compose one of these or subclass `Eval` directly. Subclassing is straightforward: implement `eval_sample(self, sample, rng)` and call `evals.metrics.get_accuracy(...)` (or your own metric) inside it. The framework handles concurrency, retries, and record logging.

## Model-graded evals: the meat

The distinctive feature of OpenAI evals is the `ModelBasedClassify` machinery, which factors a model-graded eval into three pieces:

1. **The task prompt.** Whatever prompt the *subject model* (the one being evaluated) sees to produce a response.
2. **The rubric prompt.** Whatever prompt the *grader model* sees, populated with the task, the subject's response, and (usually) a reference answer. The grader is asked to output a categorical verdict.
3. **The choice map.** A YAML mapping from the grader's categorical output (e.g. `"A"`, `"B"`, `"C"`) to a numeric score (e.g. `1.0`, `0.5`, `0.0`).

The rubric is declared in a `modelgraded/<name>.yaml` file under the registry:

```yaml
# modelgraded/sentiment_grader.yaml
prompt: |
  You are an expert grader. A task classifier was asked to label a customer email
  as one of: positive, neutral, negative. You will judge whether the classifier's
  answer is correct.

  Email:
  """
  {input}
  """

  Correct label: {ideal}

  Classifier's answer:
  """
  {completion}
  """

  Was the classifier's answer correct? Answer with a single letter:
  A) Yes, the classifier's answer matches the correct label (allowing for reasonable paraphrase).
  B) No, the classifier's answer does not match the correct label.

  Answer:

choice_scores:
  A: 1.0
  B: 0.0

choice_strings: ["A", "B"]
input_outputs:
  input: completion
```

And bound in the eval YAML via `class: evals.elsuite.modelgraded.classify:ModelBasedClassify`:

```yaml
customer_sentiment.mg.v0:
  class: evals.elsuite.modelgraded.classify:ModelBasedClassify
  args:
    samples_jsonl: customer_sentiment/samples.jsonl
    eval_type: cot_classify
    modelgraded_spec: sentiment_grader
    modelgraded_spec_args:
      completion_sample_templates: {}
```

Running this eval with `oaieval gpt-4o-mini customer_sentiment.mg.v0 --modelgraded_completion_fn gpt-4o` uses `gpt-4o-mini` as the subject and `gpt-4o` as the grader.

## Designing a rubric that survives contact with real responses

A model-graded eval is only as good as its rubric. Four rules of thumb, each earned from watching rubrics fail in production.

**Constrain the output space.** The grader must be forced to emit a bounded categorical output, not free text. `"Answer with a single letter: A, B, or C."` A grader that can say `"Well, technically the answer is somewhat correct but..."` is a grader you cannot aggregate.

**Include the reference in the rubric.** Do not ask the grader to independently know the correct answer — put it in the prompt. `Correct label: {ideal}` is the rubric's ground truth. This factors out the grader's own knowledge and reduces judgment to comparison, which graders do reliably; open-domain judgment, which they do poorly.

**Prefer few coarse categories over many fine ones.** Three levels (`fully_correct` / `partially_correct` / `incorrect`) is more reliable than a 1–10 numeric scale. The categorical → numeric mapping in `choice_scores` gives you the continuous score for aggregation without asking the grader to make finer distinctions than it can.

**Chain of thought before the verdict.** `cot_classify` prompts the grader to reason first, then extract the final letter. This routinely moves grader-human agreement by 5–15 points on non-trivial rubrics. The cost is a longer grader response; on GPT-4-class graders this is worth it.

Not all rubrics that seem to work in a small pilot survive the full eval. The next section is about how you check.

## Validating the grader against human labels

A model-graded eval is unusable if you cannot show that the grader agrees with humans on your task. The validation protocol:

1. **Human-label a subset.** Sample 100–300 items from your eval, uniformly across expected task difficulty (not just the easy ones). Have 2–3 humans label them; adjudicate disagreements (mod-102 Chapter 4). This is your **grader-validation set** — a set with trustworthy human labels, held out from any grader-tuning.
2. **Run the eval on those items.** Collect the subject model's response and the grader's verdict for each.
3. **Compute grader-vs-human agreement** on the validation set. Cohen's κ for the categorical verdict, or Pearson / Spearman correlation if the score is continuous. Report accuracy of the grader (fraction of items where the grader's verdict matches the adjudicated human label) alongside κ, because κ is prevalence-sensitive.
4. **Decide the pass/fail threshold.** A grader with `κ ≥ 0.7` on your task is usable for high-stakes reporting; `0.5 ≤ κ < 0.7` is usable with caveats; `κ < 0.5` is not usable without rubric revision. These thresholds are not universal — a low-stakes formatting eval can tolerate lower κ than a high-stakes safety eval.
5. **If the grader fails, revise the rubric.** Common fixes: add more explicit criteria, break the categorical space into finer buckets, add examples of borderline cases, switch to a stronger grader model. Do *not* keep re-running against the same validation set until you get a pass — that overfits your rubric to the validation set. If you revise more than once, cut a new validation set.
6. **Report the grader-human agreement number alongside the eval number** in every downstream report. A subject-model score without a grader-agreement number is a claim about the grader's opinions dressed up as a claim about the subject model.

The tempting shortcut — "trust the grader, it's a big model" — reliably fails. Graders have systematic biases: they favor longer responses (Zheng et al. 2023), they favor responses that mimic the grader's own house style, they defer to responses that reason step-by-step even when the reasoning is wrong. mod-105 covers the LLM-as-judge pathology landscape in depth; the validation set is the mechanism that catches these biases before they show up in your reported number.

## Building an end-to-end task: customer sentiment, graded

Putting Chapter 2's task into OpenAI evals with a graded rubric.

**Samples file** (`my_registry/data/customer_sentiment/samples.jsonl`, one line per item):

```json
{"input": [{"role": "system", "content": "Classify the sentiment of the customer email."}, {"role": "user", "content": "The product arrived broken..."}], "ideal": "negative"}
```

**Registry entry** (`my_registry/evals/customer_sentiment.yaml`):

```yaml
customer_sentiment:
  id: customer_sentiment.mg.v0
  description: Three-way sentiment classification, model-graded.
  metrics: [accuracy]

customer_sentiment.match.v0:
  class: evals.elsuite.basic.match:Match
  args:
    samples_jsonl: customer_sentiment/samples.jsonl

customer_sentiment.mg.v0:
  class: evals.elsuite.modelgraded.classify:ModelBasedClassify
  args:
    samples_jsonl: customer_sentiment/samples.jsonl
    eval_type: cot_classify
    modelgraded_spec: sentiment_grader
```

**Rubric** (`my_registry/modelgraded/sentiment_grader.yaml`, as shown above).

Run all three (exact-match, and model-graded with two different graders):

```bash
# Cheap-and-dirty exact match — a floor
oaieval gpt-4o-mini customer_sentiment.match.v0 --registry_path ./my_registry

# Graded with a small grader
oaieval gpt-4o-mini customer_sentiment.mg.v0 \
  --registry_path ./my_registry \
  --modelgraded_completion_fn gpt-4o-mini

# Graded with a strong grader
oaieval gpt-4o-mini customer_sentiment.mg.v0 \
  --registry_path ./my_registry \
  --modelgraded_completion_fn gpt-4o
```

Then run the validation-set protocol above with the two graders and pick the one whose κ against humans is highest, subject to cost budget.

## The record log

The `--record_path` output is a JSONL event log: one line per event, with events for `sampling` (a model call), `match` (a scorer verdict), and `metrics` (aggregates). Reconstructing a per-sample view requires joining events on `sample_id`. The evals repo ships `evals.data.log_events` helpers and `evals-cli`-adjacent scripts for common queries.

Two must-have queries:

- **Per-sample subject-vs-grader disagreement.** Where did the grader think the subject was wrong? Sample a handful and re-read the rubric; this is how you catch a rubric that is not doing what you meant.
- **Grader latency and cost.** The grader is often the more expensive model. `--modelgraded_completion_fn` cost per 1k items is a real budget line. If it exceeds the subject-model cost by an order of magnitude, consider a cheaper grader with validation.

## When to use (and not use) OpenAI evals

**Use when.** Open-ended generation you need to score with a rubric. The task is single-turn or lightly multi-turn. You want to ship a rubric alongside the eval and let others run it against their own models. The community has already published a rubric close to what you need — the OpenAI evals repo has many; forking is faster than authoring.

**Do not use when.** You need log-likelihood ranking (use lm-eval). You need multi-turn tool-using solvers or a sandbox (use Inspect). You need a multi-scenario matrix report with efficiency and robustness metrics (use HELM). You are running a purely programmatic exact-match eval on a chat model with a single tokenizer — lm-eval's `output_type: generate_until` will do the job with less ceremony.

## Summary

OpenAI evals decomposes an eval into a YAML registry entry plus a JSONL samples file, run by `oaieval` against a `CompletionFn` model adapter. Its distinctive feature is the `ModelBasedClassify` machinery — first-class model-graded scoring with a declarative rubric and a categorical-to-numeric choice map. Rubrics that survive contact with real responses constrain the grader to a bounded categorical output, include the reference answer in the prompt, prefer few coarse categories, and use chain-of-thought before the verdict. Every model-graded eval must be validated against a human-labelled subset with Cohen's κ or an equivalent agreement statistic; a subject-model score without a grader-agreement number is unpublishable. OpenAI evals is the right harness for rubric-graded open-ended generation and the wrong one for log-likelihood MC, agent-style tool use, or holistic multi-metric reports. Chapter 7 closes the loop with the reproducibility protocol that ties every harness in this module together: prompt format, decoding, seeds, hashes, and the discipline of pinning them.
