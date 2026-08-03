# exercise-01: Register a Custom Task in lm-evaluation-harness (Log-Likelihood + Generation)

**Estimated effort:** 4 hours

## Objective

Register a new task in EleutherAI's lm-evaluation-harness end to end, in **two variants** on the same underlying dataset: a log-likelihood (multiple-choice) evaluator and a generation-and-parse evaluator. Run both against a local or served open-weights model, and write a short report characterizing the per-item agreement between the two evaluators. The goal is not to invent a novel benchmark — it is to force through the full harness workflow (YAML registration → run → per-sample log inspection → gap analysis) at a level where a reviewer could rerun your task from your MANIFEST.

## Prerequisites

- mod-104 Chapters 1–3 (harness anatomy, lm-eval task YAML, log-likelihood vs. generation).
- mod-101 Chapters 3–4 (Wilson CIs, bootstrap CIs).
- A working install of `lm-eval` (v0.4.x). See the [EleutherAI lm-evaluation-harness README](https://github.com/EleutherAI/lm-evaluation-harness) for install steps.
- Access to at least one small open-weights model runnable locally or on a served OpenAI-compatible endpoint. Any of Llama-3-8B(-Instruct), Mistral-7B(-Instruct), Qwen2-7B(-Instruct), or Gemma-2-9B(-Instruct) is fine; a smaller model (e.g. Phi-3-mini) works if you are GPU-constrained.

## Dataset (pick one)

Pick a dataset with:
- a **small, enumerable label space** (2–5 classes) — this is required for the multiple-choice variant,
- **short input text** so log-likelihood scoring is cheap,
- **publicly available** with a canonical HF Hub path,
- **not one of the "core" lm-eval tasks** (skip MMLU / HellaSwag / ARC / TruthfulQA / BoolQ so you cannot just crib their YAMLs).

Reasonable choices:

- **`ag_news`** (`fancyzhx/ag_news` or `SetFit/ag_news`). 4-way topic classification (World / Sports / Business / Sci-Tech). ~7,600 test items.
- **`SetFit/sst5`** — 5-way sentiment (very negative → very positive). ~2,200 test items.
- **`nyu-mll/glue` config `sst2`** — binary sentiment, ~1,800 test items.
- **`emotion`** (`dair-ai/emotion`) — 6-way emotion classification (joy, sadness, anger, fear, love, surprise).
- **`banking77`** (`PolyAI/banking77`) — 77-way intent classification. Larger label space; only pick this if you want to stress-test the log-likelihood evaluator on many candidates.
- **An internal, publicly-shareable dataset** of yours with the same properties.

Cap the test-set size to ≤ 500 items if compute is tight (via `--limit 500` at run time). Larger is better for CI width but not required for correctness.

## Requirements

### Part A — MANIFEST first

Before touching YAML, write `MANIFEST.md` capturing the pins you will use (see Chapter 7). At minimum:

- Harness version (git commit hash and `pip show lm-eval`).
- Model identity (HF repo + revision).
- Dataset (HF repo + revision — pin the revision).
- Decoding config for the generation variant (should be deterministic for classification).
- Seed and few-shot config (number of shots, split, seed).
- Environment (`pip freeze` snapshot).
- Hardware.

The manifest is a live document; update it as you make choices.

### Part B — the log-likelihood (multiple-choice) task

Write `tasks/<taskname>/mytask_mc.yaml` with `output_type: multiple_choice`. Requirements:

- Enumerate the label strings in `doc_to_choice` in a canonical order and document that order in a comment.
- `doc_to_target` returns the index into `doc_to_choice`.
- `metric_list` includes both `acc` and `acc_norm`.
- `num_fewshot` is set to a value drawn from the dataset's `train` or `validation` split, with a seed.
- Prompt template ends with **no trailing whitespace**; if the target needs a leading space, put it in the choices (see Chapter 2).

### Part C — the generation task

Write `tasks/<taskname>/mytask_gen.yaml` with `output_type: generate_until`. Requirements:

- Prompt template instructs the model to respond with exactly one label from the enumerated set.
- `generation_kwargs` is deterministic (`do_sample: false`, small `max_gen_toks`, a stop sequence such as `"\n"`).
- A `filter_list` extracts the label from the raw output (regex over the label set, then `take_first`).
- `metric_list` includes `exact_match` with `ignore_case: true`.

### Part D — a group file

Write `tasks/<taskname>/mytask.yaml` that groups both variants under one name so a single command runs both:

```yaml
group: mytask
task:
  - mytask_mc
  - mytask_gen
```

### Part E — run both against a chat-tuned and a base variant of the same family (if available)

Run the group against **at least one** model, and **preferably two variants of the same model family** (e.g. Llama-3-8B and Llama-3-8B-Instruct). The point is to see the log-likelihood-vs-generation gap change direction across the base / chat pair.

Concrete run command shape (adapt to your model backend):

```bash
lm_eval \
  --model hf \
  --model_args pretrained=<repo>,dtype=bfloat16 \
  --include_path ./tasks/mytask \
  --tasks mytask \
  --num_fewshot 4 \
  --batch_size auto \
  --apply_chat_template \
  --fewshot_as_multiturn \
  --output_path runs/mytask/<model_id> \
  --log_samples \
  --seed 1234
```

For chat-tuned models use `--apply_chat_template --fewshot_as_multiturn`. For base models, drop those flags. Log the exact command in the MANIFEST.

### Part F — the gap-analysis report

Write `REPORT.md` (≤ 2 pages) containing:

1. **A summary table** with, per (model, task-variant), the metric point estimate and 95% CI:

   | model | variant | metric | value | 95% CI |
   |---|---|---|---|---|
   | ... | mc | acc | 0.712 | (0.688, 0.735) |
   | ... | mc | acc_norm | 0.751 | (0.727, 0.774) |
   | ... | gen | exact_match | 0.663 | (0.639, 0.687) |

   Use lm-eval's built-in bootstrap standard error as the CI or compute your own Wilson interval for accuracy from `n_correct` and `n_total`.

2. **A per-item agreement analysis.** Join the two variants' `--log_samples` JSONL files on `doc_id`. Compute:
   - The fraction of items where both variants agree on the label the model predicted.
   - A 2×2 confusion table: `mc_correct × gen_correct`.
   - The number of items where mc got it right and gen got it wrong, and vice versa.

3. **A qualitative slice.** Sample 5–10 items where the two variants disagreed and read the raw prompts and responses. Categorize the disagreements — chat-tuned style verbosity? Length-bias in log-likelihood? Filter regex failure? Each disagreement gets a one-line note in the report.

4. **A short discussion** (1–2 paragraphs). Which evaluator you would ship as the "headline" for this task and why. Reference the four gap-source axes from Chapter 3 / Chapter 4.

### Part G — reproducibility check

The submission must include a `run.sh` (or equivalent) that reruns the eval end-to-end from a clean working tree given the MANIFEST-pinned versions, producing byte-identical `results.json` files.

## Starter guidance

- **Do not skip the `--log_samples` flag.** Almost every question about a score can only be answered from the per-sample log; without it you are guessing.
- **Sanity-check the rendered prompt.** After the first run, open the samples JSONL and grep for the first item. Look at the exact prompt the model saw, whitespace and newlines included. This is where "why is my accuracy random" bugs live.
- **On a chat-tuned model, forgetting `--apply_chat_template` will systematically deflate the score by 10–30 points.** If your generation variant is much worse than expected, this is usually why.
- **`banking77`'s 77-way MC will hit context limits** on some models when combined with a few-shot prompt. If you picked it, keep `num_fewshot` low (0–2) or truncate carefully; document the choice.
- **For the generation variant's filter regex, alternate the labels in ranked order of length** (longest first). `(very negative|negative|neutral|positive|very positive)` will greedily match `negative` before `very negative` unless you put `very negative` first.
- **When comparing per-item disagreements, the join key is `doc_id`** in lm-eval's samples log. If your dataset does not have a natural ID, lm-eval assigns row indices.

## Acceptance criteria

- MANIFEST.md contains every pin from the Chapter 7 checklist and matches the `run.sh`.
- Both YAML files register cleanly (`lm_eval --tasks mytask --tasks_list` shows both variants).
- `results.json` is produced for both variants and includes bootstrap standard errors for every metric.
- Every metric in REPORT.md's summary table has a 95% CI; no bare point estimates.
- The per-item agreement table is present, correctly derived from the per-sample logs, and mathematically consistent (the four cells sum to n_items).
- At least one qualitative disagreement analysis identifies a mechanism (verbosity, length bias, filter failure) and cites the specific item.
- The discussion picks a headline evaluator with reasoning that references Chapter 3.
- `run.sh` reproduces the numeric results to within floating-point noise on the same hardware.
- Chat-template usage is documented and matches the model type (chat → template on; base → template off).

## Stretch goals

- **Sensitivity sweep.** Register three variants of the log-likelihood task differing only in prompt template (e.g. `"Q:/A:"`, `"Question:/Answer:"`, `"### Question/### Answer"`). Report the per-model score spread across the three. This is a lightweight preview of exercise-05.
- **Normalisation comparison.** For the log-likelihood variant, additionally register a PMI-normalized variant (`log P(target | context) - log P(target | "")`). This requires a custom metric or a post-processing script; report the per-model impact.
- **Cross-model ranking robustness.** If you ran two models, add a `winner_by_variant` column: for each task-variant pair, name the winning model. If the ranks disagree across variants, the pair-wise comparison is not robust; discuss.
- **Serve the model via vLLM and rerun through `local-chat-completions`.** Confirm your log-likelihood numbers are *not* affected by the backend swap (they should not be — same model, deterministic scoring). Any drift is a bug in one of the two paths.
