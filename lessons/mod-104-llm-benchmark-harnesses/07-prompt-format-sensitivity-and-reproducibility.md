# Prompt-Format Sensitivity and the Per-Task Reproducibility Protocol

Every previous chapter in this module has ended with a warning that "the harness version, task template, decoding, and dataset revision have to be pinned or the number is not reproducible." This chapter is where that warning becomes a protocol. It also names the shared pathology the four harnesses cannot design their way out of: **LLM benchmark scores are more sensitive to prompt formatting than to almost any modelling change** you would consider making. Sclar et al. (2024) showed that swapping between semantically equivalent prompt templates on MMLU moves per-model accuracy by up to 76 points, and reranks model leaderboards. Alzahrani et al. (2024) showed the same for changes as small as reordering choice letters. Mizrahi et al. (2024) generalized: prompt-template variance is comparable in magnitude to the model-size effect the benchmark is trying to measure. If you are not measuring your task's prompt sensitivity, your reported number's error bars are wrong.

This chapter walks the sources of prompt-format sensitivity, the log-prob-vs-generation gap you now know from Chapter 3 as one specific case of it, and the reproducibility protocol — chosen so it composes across the four harnesses.

## Sources of prompt-format sensitivity

Six knobs move the score without changing what the eval is nominally measuring. Every serious eval writeup should have inspected each.

**1. The prompt template itself.** `"Question: {q}\nAnswer:"` vs. `"Q: {q}\nA:"` vs. `"### Question\n{q}\n\n### Answer"` vs. `"[INST] {q} [/INST]"`. Sclar et al. tested 10 syntactic variations on MMLU and found accuracy spreads of 5–76 points *per model* across the variations. There is no universal "best" template — different templates favor different models — and small changes (colon vs. no colon, single newline vs. double, `A.` vs. `(A)` vs. `[A]` before choices) are each worth a few points.

**2. Choice ordering and choice-letter mapping.** For multiple-choice tasks, permuting the order of choices (A, B, C, D → C, A, D, B) changes scores because models have position priors (they prefer certain letter slots when uncertain). Alzahrani et al. quantified this: some models drop 5+ points from a random shuffle of choice positions.

**3. Few-shot demonstration selection and ordering.** The choice of *which* training items to show as demonstrations, and the order in which they are shown, changes accuracy by several points. Some tasks are stable ("random 5-shot from `train`"); others depend on the demonstration selection strategy (max-marginal, class-balanced). Fixing the demonstrations (same items, same order) with a fixed seed is the reproducibility floor; some published numbers depend on the specific demonstration set the harness happens to pick with `seed=42`.

**4. Chat template vs. completion template.** Applying a chat model's template around the eval prompt (system + user turns) versus feeding the model a bare completion-style prompt can move scores 5–15 points. The direction depends on the model: chat-tuned models usually benefit from the template; base models are indifferent or worse. Chapter 2 called this out; it is the single most common "why is my rerun wrong" bug.

**5. Tokenization edge cases.** Trailing whitespace at the prompt boundary. Whether the `bos_token` is prepended. Whether the tokenizer strips leading spaces on the target for log-likelihood scoring. Different sentencepiece / BPE tokenizers handle these differently, and a change in tokenizer library version (transformers pinning to a new sentencepiece) can move scores without you touching the model or the task.

**6. Decoding parameters.** Greedy (`T=0`, `do_sample=false`) vs. sampled (`T=0.7`, `top_p=0.95`). Max generation length. Stop-sequence list. On GSM8K in particular, truncating the model's chain-of-thought too early converts correct answers into wrong ones; a max-length change alone can move a GSM8K number by 5+ points.

The log-prob-vs-generation gap from Chapter 3 is a *seventh* source, and it interacts with all six above (a template that favors log-likelihood ranking may hurt generation and vice versa).

## Measuring prompt-format sensitivity: the diagnostic report

The remedy for prompt-format sensitivity is not "pick the best prompt" (there isn't one). It is to measure the spread and report it. The diagnostic report answers "how much of my headline number is task signal, and how much is prompt-plumbing noise?"

The recipe:

1. **Enumerate the axes you will vary.** At minimum: template variant (5–10 semantically equivalent variants), choice-order permutation (for MC tasks), few-shot demonstration seed (3–5 seeds), and — if applicable — chat-template on/off. This is a 4-dimensional design; you do not need the full factorial. A one-at-a-time (OAT) sweep from a chosen baseline is a defensible cheap version.
2. **Run the eval under each cell.** Use lm-eval's `--tasks task1,task2,...` with one YAML variant per template, or write a light shell wrapper that varies `--seed`. Under Inspect, the solver chain lets you parameterize the template as a solver argument. Under HELM, each variant is a separate `RunSpec`. Under OpenAI evals, samples files with different prompt renderings.
3. **Aggregate as a distribution, not a point.** Report the median score across variants and the 5th–95th percentile spread. If the spread is wider than the score difference between your candidate models, no shipping decision can be based on a single-variant score.
4. **Report the variant that produced each extreme.** If the highest score came from `"### Question\n{q}\n\n### Answer:"` and the lowest from `"[INST] {q} [/INST]"`, the report says so and lets a reader reason about which one matches production usage.
5. **Include a rank-stability check.** If you are comparing multiple models, do their relative ranks stay the same across variants? If Model A beats Model B under variant 1 but loses under variant 5, the comparison is not robust — report the frequency-of-winning across variants alongside the point estimates.

Exercise 5 is exactly this diagnostic on a single task.

## The per-task reproducibility protocol

Any eval that is going to be re-run — by you next quarter, by a reviewer, by another team — needs a reproducibility pinning. The protocol below is what every serious LLM eval writeup already implicitly relies on; naming it lets you audit whether your writeup actually meets the bar.

### The MANIFEST

Every eval run emits a `MANIFEST.md` (or `manifest.json`) capturing exactly what produced the numbers. The minimum fields:

- **Harness name and version.** `lm-evaluation-harness @ git 3a4f5b7` or `helm-instruct @ v0.5.4` or `inspect-ai==0.3.72` or `openai-evals @ git a1b2c3d`. Pin to a commit if the release cadence is loose.
- **Task name and task-file hash.** `task=customer_sentiment_gen`, `task_yaml_sha256=abc...`. If the task is registered from an out-of-tree YAML, hash the resolved config (`results.json["configs"][task_name]` from lm-eval; the equivalent from other harnesses).
- **Dataset revision.** `dataset=myorg/customer-sentiment`, `revision=abc1234def` (git-style commit hash on HF Hub, or a checksum of a local Parquet snapshot). Never leave this to "latest."
- **Model identity.** Weights hash if the model is on disk (or the HF revision hash if hosted); model ID for hosted APIs. Include quantization, precision (`bf16`/`fp16`/`int8`), and any weight-modifying options.
- **Decoding config.** Full `generation_kwargs`: `do_sample`, `temperature`, `top_p`, `top_k`, `max_gen_toks`, `stop`, and any provider-specific extras.
- **Seed.** For any sampling: `seed=1234`. Note that a seed is only reproducible on the same sampler implementation; note the sampler.
- **Few-shot config.** `num_fewshot=5`, `fewshot_split=train`, `fewshot_seed=42`, and (if the harness supports it) the actual demonstration IDs used.
- **Prompt template hash.** `sha256` of the rendered prompt for a canonical example (the first item, say). This catches silent template changes across harness versions.
- **Environment.** Python version, key library versions (`transformers`, `vllm`, `tokenizers`, `sentencepiece`, `torch`, `cuda`). `pip freeze > MANIFEST_environment.txt`.
- **Hardware.** GPU model, count, and driver. `float16` on a T4 and `bf16` on an H100 give different log-likelihoods; the record makes that observable.
- **Wall-clock and cost.** Not strictly reproducibility, but the reviewer will want it.

Every harness in this module writes *most* of these fields by default. Nobody writes all of them. The MANIFEST is the artifact you assemble from the harness output plus a few extra lines your CI or run script emits.

### The reproducibility checklist

Before signing off on an eval report, walk this checklist:

- [ ] MANIFEST captures every field above.
- [ ] `--log_samples` (or the harness equivalent) was on and the per-sample log is stored alongside `results.json`.
- [ ] Prompt-template hash is included and the actual rendered prompt for the first item is inspectable.
- [ ] Decoding is deterministic (`do_sample=false` for classification; explicit seed if sampling).
- [ ] Dataset revision is pinned.
- [ ] Harness version is pinned to a commit.
- [ ] For any comparison across models: the same task version, dataset revision, and decoding config were used for every model. Comparing runs from different harness versions is not comparing models.
- [ ] For any comparison against a published number: the published harness, version, and adapter are named, and the reproduction attempt was made under those conditions before switching to your own.
- [ ] For any model-graded score: the grader model, grader prompt, and grader-vs-human κ are included (Chapter 6).
- [ ] For any prompt-sensitive task: at least a small template variance measurement (say 3 variants) has been run and the reported spread contains the headline number.

### The dataset-hash discipline

The dataset revision is the field that most silently breaks. The failure mode: your task points at `hf-org/mytask` without a revision. The upstream fixes a typo in one example and pushes a new revision. Your rerun uses the new revision. Your accuracy moves by 0.1%. No one notices for six months, at which point a reviewer tries to reproduce the launch report and cannot.

The fix is one line: `dataset_kwargs={"revision": "abc1234"}`. For local datasets, hash the Parquet / JSONL bytes and store the hash. `sha256sum data/test.parquet` in the MANIFEST is sufficient. Every harness in this module supports it; nothing forces you to use it. It costs nothing.

### The harness-version discipline

`lm-eval @ v0.4.2` and `lm-eval @ v0.4.3` on the same YAML task will typically produce different numbers for the reasons this chapter has been about (template tweaks, normalisation defaults). The 2024 Open LLM Leaderboard v2 rewrite explicitly re-benchmarked every model because harness changes moved the scores. Your writeup should not say "we ran lm-eval"; it should say "we ran lm-eval @ v0.4.2." A commit hash is even better than a tag.

The `configs` block that lm-eval and HELM include in their results JSON is the *resolved* task config at run time. If you want to compare two runs on the same task, diff their `configs` blocks first. Every field that differs is a candidate for the score difference; if `configs` is identical and the numbers still differ, look at the environment (library versions, hardware precision).

## The composability of the four harnesses' pins

Different harnesses expose these pins differently. Translating between them:

| Pin | lm-eval | HELM | Inspect | OpenAI evals |
| --- | --- | --- | --- | --- |
| Harness version | git hash / `pip show lm-eval` | git hash / `pip show crfm-helm` | `pip show inspect-ai` | git hash of `evals` repo |
| Task version | `results.json` `versions` + `configs` | `run_spec.json` | `Task` code hash + `.eval` log | registry YAML with `id: name.v0` |
| Dataset revision | `dataset_kwargs.revision` in YAML | `ScenarioSpec.args` (scenario-defined) | `hf_dataset(..., revision=)` | hash of `samples.jsonl` |
| Decoding config | `generation_kwargs` in YAML | `AdapterSpec` (`temperature`, etc.) | model config (`ModelConfig`) or per-solver kwargs | `--completion_args` |
| Seed | `--seed` | `--seed` in `helm-run` (per-adapter) | eval log records seed | `--seed` |
| Rendered prompt | `--log_samples` JSONL | per-instance JSON with `prompt` field | eval log per-sample `messages` | `--record_path` events, `sampling` events |

The columns are not perfectly aligned — HELM has richer per-instance pins because the scenario-adapter-metric split gives each its own config, whereas OpenAI evals leans on the registry YAML plus samples JSONL. But every harness exposes each pin *somewhere*. The reproducibility question is not "does the harness let me pin this?" but "did I actually pin it in the run script and record it in the MANIFEST?"

## A worked closing example

Take the customer-sentiment task from Chapter 2. A defensible run's MANIFEST section:

```
harness: lm-evaluation-harness @ 3a4f5b7 (v0.4.2 + 3 patches)
task: customer_sentiment_gen (my_registry, sha256=deadbeef...)
dataset: myorg/customer-sentiment @ revision=abc1234
n_test: 1000
model: meta-llama/Meta-Llama-3-8B-Instruct @ revision=e1a2b3c
precision: bfloat16
decoding: do_sample=false, max_gen_toks=8, until=["\n"]
num_fewshot: 4 (fewshot_split=validation, fewshot_seed=42, IDs=[v_0004, v_0071, v_0138, v_0199])
prompt_template_hash: 9f8e7d6c...
chat_template: applied (fewshot_as_multiturn=true)
seed: 1234
env: python 3.11.8, transformers==4.44.0, tokenizers==0.19.1, torch==2.4.0+cu121
hw: 1x H100 80GB, driver 550.90.07
```

Every field is one line. Together they are the run.

Now imagine six months later a reviewer wants to check that a claimed improvement is real. They read this MANIFEST, install the harness at the pinned commit, install the pinned library versions, pull the pinned dataset revision, run the same command, and get the same number to within floating-point noise. That is what reproducibility looks like when it works. It requires no extra tooling — every field above comes from either a CLI flag you already passed, a config block the harness already emits, or a `pip freeze` you can capture in three seconds.

## What the discipline is for

The point of the reproducibility protocol is not that anyone will actually rerun every eval you produce. Most numbers will stand on the trust the MANIFEST buys. The point is that when a number gets challenged — because it was surprising, because a competing team disagrees, because a regulator asks for evidence — you can go back to the MANIFEST, rerun the eval, and produce the same number. Without that ability, your eval is folklore.

Every previous chapter has ended by naming the next harness. This chapter closes the module by naming the shared discipline: **pin the pins, measure the sensitivity, report the spread, and inspect the log.** mod-105 opens the next question — what happens when the scorer is itself an LLM at scale — but everything in that module still rides on the reproducibility protocol above. If the underlying eval is not reproducible, no scorer sophistication saves it.

## Summary

LLM benchmark scores are more sensitive to prompt formatting than to almost any modelling change: template variance, choice ordering, few-shot selection, chat-template application, tokenization edges, and decoding config each move scores by several points, and together can flip model rankings. The remedy is to measure the spread and report it — median score with a 5th–95th percentile band across template variants — not to pretend a single-variant number is definitive. The per-task reproducibility protocol pins the harness version (to a git commit), the task version and its resolved config, the dataset revision, the model checkpoint and precision, the decoding config, the seed, the few-shot selection, the prompt-template hash, the environment, and the hardware. Every harness in this module surfaces these pins somewhere; the discipline is to capture them in a MANIFEST and to run the checklist before signing off. Two harnesses' "same" task will not agree, and no protocol changes that; but within a harness, at a pinned version, on a pinned dataset, with a pinned decoding config, an eval number is reproducible. That is what makes it possible to argue about, defend, or challenge — which is the whole point of running an eval.
