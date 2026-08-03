# EleutherAI lm-evaluation-harness: Registering and Running a Task

EleutherAI's [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) (`lm-eval` in the CLI, "lm-eval-harness" or "harness" in prose) is the de-facto reference harness for reproducible LLM benchmarks. The Hugging Face Open LLM Leaderboard runs on it. Papers reporting MMLU / HellaSwag / ARC / TruthfulQA / GSM8K usually mean "at the version of lm-eval that shipped with the model release." If your job is to run a new task against many models, or to reproduce a published score to defend or dispute it, you will spend more of your working hours in this harness than any other. This chapter walks through the task-registration model — the YAML surface that turns a dataset into a runnable task — and gets you to a first end-to-end run.

## The mental model

lm-eval is built on four abstractions that map cleanly onto the four objects from Chapter 1.

- **`LM` (the model adapter).** A Python class implementing `loglikelihood`, `loglikelihood_rolling`, and `generate_until`. The distributed set includes `HFLM` (transformers), `HFLM` in accelerate/vLLM mode, `openai-chat-completions`, `openai-completions`, `anthropic-chat`, `local-chat-completions` (any OpenAI-compatible endpoint including vLLM, TGI, TensorRT-LLM), `mamba`, `neuralmagic`, and a growing list. Which adapter you pick is passed via the `--model` and `--model_args` CLI flags.
- **`Task` (the task definition).** Historically a Python class; since v0.4 the recommended path is a YAML file that gets loaded by the `ConfigurableTask` machinery. The YAML declares the dataset, the prompt rendering, the request type, and the metrics.
- **`Instance` (a request).** The harness materializes each dataset item into one or more `Instance` objects, each with a request type (`loglikelihood`, `loglikelihood_rolling`, `generate_until`) and its arguments. This is what actually gets batched and sent to the model.
- **`Metric`.** A per-item or per-corpus scoring function, typically referenced by name (`acc`, `acc_norm`, `exact_match`, `perplexity`, `bleu`, `mc1`, ...) from the built-in registry, but user-definable in Python.

The unit of work you register is a **task**. A **group** is a bundle of tasks (e.g. `mmlu` is a group whose tasks are the 57 subjects). Tasks can inherit from other tasks via YAML `include:`, which is how variants (chain-of-thought, cloze-style, generative) share configuration.

## The YAML you actually write

A minimal task file. Save as `lm_eval/tasks/mytask/mytask.yaml` (or point at it with `--include_path`):

```yaml
task: mytask
dataset_path: hf-org/mytask
dataset_name: null            # subset name if the HF dataset has multiple configs
test_split: test
fewshot_split: train          # where to sample few-shot demonstrations from
output_type: multiple_choice  # or loglikelihood_rolling, or generate_until
doc_to_text: "Question: {{question}}\nAnswer:"
doc_to_target: "{{answer_idx}}"     # for multiple_choice: index into doc_to_choice
doc_to_choice: "{{[choice_a, choice_b, choice_c, choice_d]}}"
metric_list:
  - metric: acc
    aggregation: mean
    higher_is_better: true
  - metric: acc_norm
    aggregation: mean
    higher_is_better: true
num_fewshot: 5
```

The template strings are Jinja2 with the dataset row as the template context — every column of the row is a variable. `doc_to_choice` returns a list; the harness will issue one `loglikelihood` request per choice and pick the argmax. `doc_to_target` is the *index* into the choices list for multiple-choice tasks; for generative tasks it is the *string* to score against.

The four fields that carry all of the task's semantics are `output_type`, `doc_to_text`, `doc_to_target`, and `metric_list`. Get those four right and the rest of the YAML is plumbing.

### Prompt rendering: `doc_to_text` and `doc_to_target`

The prompt that reaches the model is (roughly):

```
{fewshot_1_text}{fewshot_1_target}\n\n
{fewshot_2_text}{fewshot_2_target}\n\n
...
{test_doc_text}
```

The harness concatenates few-shot demonstrations before the test item, using the same `doc_to_text` and `doc_to_target` templates. The concatenator (`fewshot_delimiter`, default `"\n\n"`) and the number of shots (`num_fewshot`) are configurable. If the model backend applies a chat template, that template wraps the *entire* concatenated string, unless you explicitly opt into per-turn few-shot with `apply_chat_template` and `fewshot_as_multiturn`.

Two subtle failure modes.

- **Trailing whitespace in `doc_to_text`.** Whether your prompt ends with `Answer:` or `Answer: ` changes tokenization at the boundary and changes log-likelihood scores. The convention in most published tasks is *no* trailing space; the space is part of the target (so `doc_to_choice` values start with a leading space: `" A"`, `" B"`, ...). Get this wrong and the scores drift a percent or two, invisibly.
- **Newline handling.** Jinja2's `{{ ... }}` does not add trailing newlines, but multi-line templates in YAML often gain or lose a newline through YAML block-style vs. flow-style conversion. Use `|` (literal) or `>` (folded) block scalars explicitly if the prompt shape matters, and inspect the rendered string with `--log_samples` before you trust a score.

### `output_type` and what it implies

Four values matter in practice:

- **`multiple_choice`** — the harness issues one `loglikelihood` request per candidate in `doc_to_choice`, picks the argmax, and scores against `doc_to_target` (an integer index). Reports both `acc` (raw log-likelihood argmax) and `acc_norm` (log-likelihood normalized by the byte-length of the choice). See Chapter 3.
- **`loglikelihood`** — the harness issues one `loglikelihood` request scoring the target against the context and reports the log-likelihood directly (or a derived metric like `perplexity`). Used for LM benchmarks (WikiText, LAMBADA-style).
- **`loglikelihood_rolling`** — used for perplexity over long documents; slides a window across the full document.
- **`generate_until`** — the harness calls `generate_until` on the adapter with configurable `stop`, `max_gen_toks`, `do_sample`, `temperature`, and `top_p`, and passes the generated string to the metric functions.

Multiple-choice tasks *can* be run either way (log-likelihood ranking or generation-of-a-letter); Chapter 3 is entirely about when the two disagree and which one to trust.

### `metric_list`

Each entry names a metric, its per-item form, and its aggregation. Common built-ins:

| metric | request type | notes |
| --- | --- | --- |
| `acc` | `multiple_choice` | raw log-likelihood argmax accuracy |
| `acc_norm` | `multiple_choice` | length-normalized argmax |
| `exact_match` | `generate_until` | with configurable normalization (whitespace, punctuation, articles) |
| `f1` | `generate_until` | SQuAD-style token F1 |
| `perplexity` | `loglikelihood_rolling` | exp of mean NLL per token |
| `bleu`, `rouge` | `generate_until` | via `sacrebleu` / `rouge_score` |
| `mc1`, `mc2` | `multiple_choice` | TruthfulQA-style; `mc2` sums normalized probability mass over all correct answers |

You can register a custom metric by pointing at a Python callable (`metric: !function utils.my_metric`) — the callable receives predictions and references and returns a numeric score. Aggregation over items is by default `mean`; other options include `median`, `matthews_corrcoef`, and a user-supplied callable.

### Filters (post-processing the generation)

For `generate_until` tasks, the raw model output usually needs to be canonicalized before it can be compared to the target. Filters do this. Common examples:

```yaml
filter_list:
  - name: "strict-match"
    filter:
      - function: "regex"
        regex_pattern: "The answer is (\\-?[0-9\\.\\,]+)"
      - function: "take_first"
```

The filter chain runs per-item; each step transforms the model's raw output. The `regex` step extracts a capture group; `take_first` picks the first match. The most common cause of a "the score is zero even though the model gets it right" bug is a filter mismatch — the model wrote `"answer: 42"` and your filter looked for `"The answer is 42"`. Always inspect a handful of `--log_samples` before trusting a generative score.

## Running the task

The CLI shape is:

```bash
lm_eval \
  --model hf \
  --model_args pretrained=meta-llama/Meta-Llama-3-8B,dtype=bfloat16 \
  --tasks mytask \
  --num_fewshot 5 \
  --batch_size auto \
  --output_path runs/mytask/llama3-8b \
  --log_samples \
  --seed 1234
```

Under the hood: the model backend spins up, the task is materialized, requests are batched to fit the effective batch size (auto tunes based on OOM feedback), responses are collected, metrics are computed, and a JSON result plus a per-sample JSONL (with `--log_samples`) is written to `--output_path`.

Two model-backend patterns you will use often:

- **Local Hugging Face model.** `--model hf --model_args pretrained=<repo_or_path>,dtype=bfloat16,parallelize=True`. For inference speed on multi-GPU, use `--model vllm --model_args pretrained=<repo>,tensor_parallel_size=4`.
- **OpenAI-compatible endpoint** (works for a locally-hosted vLLM / TGI / OpenAI itself). `--model local-chat-completions --model_args base_url=http://127.0.0.1:8000/v1,model=my-served-model,num_concurrent=32`.

For chat-tuned models, add `--apply_chat_template --fewshot_as_multiturn`. Without those flags, the harness pastes few-shot demos into a single completion-style prompt, which most chat-tuned models handle poorly and which will silently deflate your score.

## The results and sample logs

Two artifacts. `results.json` in `--output_path` contains the aggregate metrics with bootstrap standard errors:

```json
{
  "results": {
    "mytask": {
      "acc,none": 0.734,
      "acc_stderr,none": 0.008,
      "acc_norm,none": 0.751,
      "acc_norm_stderr,none": 0.008
    }
  },
  "configs": {"mytask": { ... entire resolved YAML ... }},
  "versions": {"mytask": "Yaml"},
  "git_hash": "abcd1234",
  "date": "2026-08-03T12:34:56Z"
}
```

Log every field. `git_hash`, `configs`, and `versions` are what let a reproducer land on the same numbers.

The per-sample JSONL from `--log_samples` contains, per dataset item, the fully-rendered prompt, all requests, all responses, the parsed prediction, the target, and the per-item metric value. This file is the ground truth for "why did this item score zero." Never trust an aggregate you cannot reconcile against these logs.

## Registering a fully new task: end-to-end

Suppose your task is "given a short customer email, classify the sentiment as `positive` / `neutral` / `negative`." You have a Hugging Face dataset at `myorg/customer-sentiment` with columns `email`, `label` (string in `{positive, neutral, negative}`).

Write two YAML files. First the multiple-choice (log-likelihood) variant:

```yaml
# lm_eval/tasks/customer_sentiment/customer_sentiment_mc.yaml
task: customer_sentiment_mc
dataset_path: myorg/customer-sentiment
test_split: test
fewshot_split: validation
output_type: multiple_choice
doc_to_text: "Email: {{email}}\nSentiment:"
doc_to_choice: '{{["positive", "neutral", "negative"]}}'
doc_to_target: '{{["positive", "neutral", "negative"].index(label)}}'
metric_list:
  - metric: acc
    aggregation: mean
    higher_is_better: true
  - metric: acc_norm
    aggregation: mean
    higher_is_better: true
num_fewshot: 4
```

Then the generation variant, which asks the model to *write* the label and grades the extracted string:

```yaml
# lm_eval/tasks/customer_sentiment/customer_sentiment_gen.yaml
task: customer_sentiment_gen
dataset_path: myorg/customer-sentiment
test_split: test
fewshot_split: validation
output_type: generate_until
doc_to_text: "Email: {{email}}\nSentiment (one of: positive, neutral, negative):"
doc_to_target: "{{label}}"
generation_kwargs:
  do_sample: false
  max_gen_toks: 8
  until:
    - "\n"
filter_list:
  - name: "extract-label"
    filter:
      - function: "regex"
        regex_pattern: "(positive|neutral|negative)"
      - function: "take_first"
metric_list:
  - metric: exact_match
    aggregation: mean
    higher_is_better: true
    ignore_case: true
    ignore_punctuation: true
num_fewshot: 4
```

Register them by placing the files under `lm_eval/tasks/customer_sentiment/` (or a directory of your own passed via `--include_path`). Add a `group` file if you want to run both together:

```yaml
# lm_eval/tasks/customer_sentiment/customer_sentiment.yaml
group: customer_sentiment
task:
  - customer_sentiment_mc
  - customer_sentiment_gen
```

Then run:

```bash
lm_eval \
  --model hf \
  --model_args pretrained=meta-llama/Meta-Llama-3-8B-Instruct,dtype=bfloat16 \
  --include_path lm_eval/tasks/customer_sentiment \
  --tasks customer_sentiment \
  --num_fewshot 4 \
  --batch_size 16 \
  --apply_chat_template \
  --fewshot_as_multiturn \
  --output_path runs/customer_sentiment/llama3-8b-inst \
  --log_samples \
  --seed 42
```

Inspect the per-sample JSONL for both variants side-by-side. On the same underlying task, the log-likelihood-ranked and the generation-then-extracted numbers will typically not agree — that gap is the subject of Chapter 3.

## Common pitfalls when you first register a task

- **Chat template forgotten on a chat-tuned model.** The single most common "why is my score 30 points below the model card" bug. Chat-tuned models expect their template; without `--apply_chat_template`, you are feeding them base-model prompts and grading them as if they were base models.
- **Trailing whitespace on `doc_to_text` vs. leading whitespace on `doc_to_choice`.** Convention is no trailing space on the prompt, and the target string carries the leading space if any. If you set it up the other way and it works fine on one model, it may still tank a different tokenizer.
- **Sampling on for a metric that assumes determinism.** `generate_until` with `do_sample: true` and no seed makes the result non-reproducible. For classification-style generative tasks set `do_sample: false`.
- **`num_fewshot` mismatch with published numbers.** Every canonical benchmark has a published shot count (MMLU: 5, HellaSwag: 10, TruthfulQA: 0 for MC1 / 0 for MC2, GSM8K: 8, ...). Not matching it is a guaranteed non-reproduction. See Chapter 4.
- **Wrong `test_split`.** Some HF datasets ship `test` as an *unlabelled* split (competitions frequently do this). If your accuracy is inexplicably random, check that the split has ground-truth labels.
- **Metric alias mismatches.** `acc` vs. `acc_norm` differ; the "correct" number to compare against a leaderboard depends on which one the leaderboard reports. HuggingFace Open LLM Leaderboard v1 used `acc_norm` for many tasks; v2 changed several. Read the leaderboard's methodology page before comparing.

## When lm-eval is the wrong tool

lm-eval is the right tool for reproducible, log-likelihood-friendly, single-turn academic benchmarks and for adapting classification-style tasks to that mold. It is the *wrong* tool when the eval requires:

- Multi-turn tool-using interactions in a sandbox (use Inspect).
- Judge-model rubric grading with sophisticated prompts (use OpenAI evals for the rubric plumbing).
- A holistic multi-scenario report with a shared taxonomy of metrics for cross-model comparison (use HELM).

Recognizing when to leave lm-eval saves you from bending it into shapes it was not designed for.

## Summary

lm-evaluation-harness turns a dataset into a runnable, reproducible eval task through a YAML declaration of `dataset_path`, `output_type`, `doc_to_text`, `doc_to_target`, `doc_to_choice`, and a `metric_list`. The four abstractions — `LM`, `Task`, `Instance`, `Metric` — map cleanly onto the four-objects taxonomy from Chapter 1. Register a task by writing a YAML file, run it with `lm_eval --model ... --tasks ... --num_fewshot ... --log_samples`, and always inspect the per-sample JSONL before trusting the aggregate. The most common first-run bugs are whitespace at the prompt boundary, forgetting the chat template, sampling on for a metric that assumes determinism, and shot-count mismatches with published numbers. Chapter 3 opens the next problem the two variants of the customer-sentiment task from this chapter surfaced: the log-likelihood-vs-generation gap, and how to reason about which mode is the right one for a given eval.
