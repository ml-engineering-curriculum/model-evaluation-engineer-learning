# HELM Scenarios and Reproducing a Public Number

Stanford CRFM's [HELM](https://crfm.stanford.edu/helm/) — the *Holistic Evaluation of Language Models* framework — was built around the thesis that a serious language-model report is a **matrix** across many scenarios and many metrics, not a headline number. The framework's abstractions (`Scenario`, `Adapter`, `Metric`, `RunSpec`) are the cleanest published articulation of the four-objects taxonomy from Chapter 1, and its public leaderboard is the largest reproducible set of LLM eval runs in the field. This chapter walks HELM's structure, then uses reproducing a public HELM (or lm-eval) number as the concrete case for a broader skill: diagnosing why your rerun of a published benchmark did not match, and closing the gap.

## HELM's four abstractions

HELM decomposes an eval into four pieces, each replaceable.

**`Scenario`** — the dataset producer. A `Scenario` subclass implements `get_instances()`, which returns a list of `Instance` objects (an `Instance` has `input`, one or more `Reference` objects flagged with a `Correctness` tag, and a `split`). The `Scenario` owns the *raw task*: where the data comes from, how splits are made, how references are labelled.

**`Adapter`** — the prompt renderer + request formatter. An `Adapter` takes a `Scenario`'s instances plus few-shot demos and produces `Request` objects for the model. HELM ships several: `MultipleChoiceJointAdapter` (asks for `A`/`B`/`C`/`D` and generates), `MultipleChoiceSeparateAdapter` (log-likelihood over each candidate, either "original" or "calibrated" flavour), `GenerationAdapter` (open-ended generation with configurable stop tokens), `LanguageModelingAdapter` (perplexity), and a growing set for tool-using and chat evals.

**`Metric`** — the scorer. A `Metric` subclass consumes model outputs and produces named `Stat` objects (mean, count, per-slice breakdowns). HELM's built-ins cover exact match, quasi-exact match, F1, BLEU, ROUGE, perplexity, calibration, robustness, fairness, bias, toxicity, and efficiency.

**`RunSpec`** — the binding. A `RunSpec` says "run *this* `Scenario` with *this* `Adapter` and *these* `Metric`s, tagged with *these* groups." A single scenario can have many run specs (e.g. one for multiple-choice-joint, one for multiple-choice-separate, one for the calibrated variant).

The command line is `helm-run` with a `--run-specs` list; results land under `benchmark_output/`. HELM's leaderboard is populated by running the same run specs against many models and publishing the aggregated matrix, plus every request/response pair, plus the full config that produced them.

## The scenario/adapter/metric decomposition in practice

Two things this decomposition buys you.

**Portability of the scenario.** The same `Scenario` can be run under multiple adapters. HELM's public results routinely include both `mmlu:joint` (generation) and `mmlu:separate` (log-likelihood ranking) — the two evaluators from Chapter 3 — so the reader can see both numbers and reason about which one is relevant. Similarly HELM's `natural_qa:closedbook` scenario is run under a generation adapter with two normalisations of the metric (exact match and quasi-exact match).

**Composability of metrics.** A `Metric` in HELM is a *stat producer*, not a per-item scalar. Multiple metrics can consume the same responses. For a generation task you can attach `ExactMatch`, `QuasiExactMatch`, `F1`, `BLEU`, and `Efficiency` metrics to the same run and pay only one inference cost. HELM's efficiency and robustness metrics — how many tokens the model spent to reach the answer, how much accuracy dropped under paraphrasing — are computed alongside the primary metric in the same run.

The trade-off is verbosity. Defining a scenario, an adapter binding, and a run spec is more work than dropping a YAML in lm-eval. HELM pays for that verbosity with a matrix-style report that is genuinely more informative than any single number.

## A minimal HELM scenario, end-to-end

Suppose the customer-sentiment task from Chapter 2 needs a HELM run so its number lines up with other tasks on your team's HELM board.

**1. Write the scenario.** Under `src/helm/benchmark/scenarios/customer_sentiment_scenario.py`:

```python
from helm.benchmark.scenarios.scenario import (
    Scenario, Instance, Reference, TEST_SPLIT, TRAIN_SPLIT, CORRECT_TAG,
)
from datasets import load_dataset


class CustomerSentimentScenario(Scenario):
    name = "customer_sentiment"
    description = "Three-way sentiment classification of customer emails."
    tags = ["classification"]

    def get_instances(self, output_path):
        ds = load_dataset("myorg/customer-sentiment")
        instances = []
        for split_name, split in [("train", TRAIN_SPLIT), ("test", TEST_SPLIT)]:
            for row in ds[split_name]:
                refs = [
                    Reference(output=Output(text=lbl),
                              tags=[CORRECT_TAG] if lbl == row["label"] else [])
                    for lbl in ["positive", "neutral", "negative"]
                ]
                instances.append(Instance(
                    input=Input(text=row["email"]),
                    references=refs,
                    split=split,
                ))
        return instances
```

**2. Write the run spec.** In `src/helm/benchmark/run_specs.py` (or an add-on module), register a factory:

```python
@run_spec_function("customer_sentiment")
def get_customer_sentiment_spec() -> RunSpec:
    scenario_spec = ScenarioSpec(
        class_name="helm.benchmark.scenarios.customer_sentiment_scenario.CustomerSentimentScenario",
        args={},
    )
    adapter_spec = get_multiple_choice_joint_adapter_spec(
        instructions="Classify the sentiment of the email as positive, neutral, or negative.",
        input_noun="Email",
        output_noun="Sentiment",
        max_train_instances=4,
    )
    return RunSpec(
        name="customer_sentiment",
        scenario_spec=scenario_spec,
        adapter_spec=adapter_spec,
        metric_specs=get_exact_match_metric_specs(),
        groups=["customer_sentiment"],
    )
```

**3. Run it.** A single-model run spec:

```bash
helm-run \
  --run-specs customer_sentiment:model=openai/gpt-4o-mini \
  --suite my_suite_20260803 \
  --max-eval-instances 1000 \
  --output-path benchmark_output
```

HELM writes per-scenario JSON with the aggregate stats, per-instance JSONL with the full request/response trace, and a `run_spec.json` that captures the resolved config. `helm-summarize` then aggregates a `suite` (a named directory of run outputs) into a leaderboard-style table, and `helm-server` serves the interactive viewer.

**4. Run the log-likelihood variant.** Swap the adapter for the multiple-choice-separate one to get the "ranking" flavour, and register it as a second run spec:

```python
adapter_spec = get_multiple_choice_separate_original_adapter_spec(
    input_noun="Email",
    output_noun="Sentiment",
    max_train_instances=4,
)
```

You now have two HELM run specs on the same scenario, exactly analogous to the two lm-eval tasks in Chapter 3. The HELM viewer will show both side-by-side.

## Reproducing a public number: what "within tolerance" means

Now the case study half of this chapter. A published benchmark number — the Llama-3-8B row on the Open LLM Leaderboard, the GPT-4 row on the HELM classic leaderboard, a model card's MMLU value — is a claim you can test. The workflow is: (a) figure out exactly what was run, (b) run it yourself, (c) explain any gap in a written note.

**What "within tolerance" means.** Reproducibility of a benchmark number is not "identical to five decimal places." It is "inside the CI of the published number, run with the same seed on the same hardware and library versions." A defensible tolerance policy for public LLM benchmarks:

- **Deterministic runs on the same hardware and versions:** should match to within floating-point noise (typically <0.001 on a 0–1 metric). Numeric drift larger than that indicates a real change (a library upgrade, a kernel change, a different precision).
- **Deterministic runs on different hardware / different GPU generation:** typically ±0.001–0.005 on a 0–1 metric from non-associative float summation, sometimes larger for very batch-size-sensitive ops.
- **Same task, published number that used a different seed or a different subsampling:** compare against the published bootstrap CI. lm-eval reports per-task bootstrap standard errors; HELM reports standard errors and 95% CIs. If your point estimate is inside the published CI, you have reproduced; if not, you have not.
- **Sampling generation (`do_sample=true`):** never fully deterministic across implementations. Fix the seed *and* the tokenizer *and* the sampler, and expect residual noise. For decision-making eval always prefer greedy (`temperature=0` / `do_sample=false`).

The 2024 Hugging Face Open LLM Leaderboard v2 post-mortem is instructive: the maintainers documented per-benchmark score movements after harness upgrades and re-benchmarked every model. Score deltas of 1–3 accuracy points from prompt-template and normalisation changes were routine, and none were reproducibility bugs — they were the harness's own choices tightening or shifting. Pin the harness version.

## The four gap sources you will actually hit

When your rerun does not match, the discrepancy almost always comes from one of four sources. Diagnose in this order.

**1. Prompt template and few-shot.**

- **Symptom:** off by 1–10 points, consistent direction across items.
- **Diagnosis:** dump the rendered prompt for one item from your run and compare it, character by character, to the rendered prompt from the published run. Missing chat template, different `bos_token`, different few-shot demonstration selection (fixed vs. random seed), different demonstration ordering, different `system` prompt.
- **Fix:** align to the reference. For lm-eval reproducing an lm-eval public number, use the same task YAML at the same harness git hash. For lm-eval reproducing a HELM number (or vice versa), you are not really reproducing — you are running a different task that shares a name.

**2. Decoding and generation config.**

- **Symptom:** for generation-mode tasks, high variance across your reruns, or a consistent gap against a published number.
- **Diagnosis:** compare `do_sample`, `temperature`, `top_p`, `top_k`, `max_gen_toks`, and the stop-sequence list. On MATH / GSM8K in particular, published numbers assume specific `max_gen_toks` values (often 512 or 1024); truncating too early produces silent zeros.
- **Fix:** copy the published decoding config exactly. If the published number is sampled at `T > 0`, run at least 8 seeds and report the mean and 95% CI; a single sampled run is not comparable.

**3. Normalisation and metric flavour.**

- **Symptom:** off by 1–5 points, consistent, and matches suspiciously well with the size of the `acc` vs. `acc_norm` gap.
- **Diagnosis:** does the published number report `acc` or `acc_norm`? `exact_match` or `quasi_exact_match`? McCarthy F1 vs. SQuAD F1 (different tokenization rules)? On generation MC tasks, are they parsing the first letter with `re.match("[A-D]")` or `re.search(r"answer.*?([A-D])")`?
- **Fix:** read the benchmark's methodology page. The Open LLM Leaderboard, HELM, and every serious benchmark publish a "how we compute this number" section; the metric flavour is always in it. For your own tasks, the *harness version and its resolved config* should uniquely determine this — inspect `configs` in the results JSON.

**4. Dataset revision.**

- **Symptom:** off by 1–3 points with no other explanation; sample count differs from the published number.
- **Diagnosis:** the HF dataset was updated. MMLU has had at least one bug-fix rev. HellaSwag has been partially deduplicated in some copies. If your `n_test` is 14,042 and the published number used 14,036, you are not on the same dataset.
- **Fix:** pin the dataset revision. `dataset_path: hf-org/mytask` should become `dataset_path: hf-org/mytask@abc1234` (the harness supports `revision` in `dataset_kwargs`). Include the dataset hash in your MANIFEST.

If you have ruled out all four and still see a gap, the remaining candidates are library-version bugs (transformers, vLLM, sentencepiece tokenizer changes), precision differences (fp16 vs. bf16 vs. fp32 on log-likelihood computation), or actually-different model weights (a quantized checkpoint being served under an unquantized name). Log the resolved config and open an issue upstream.

## The reproduction workflow

A concrete workflow you can turn into a runbook.

1. **Pick the target.** Write down: benchmark name, exact task (e.g. `mmlu` group vs. `mmlu:humanities` subset), evaluator (log-likelihood or generation), metric (`acc` or `acc_norm`), harness name and version, model checkpoint and revision, number of few-shot examples, and the published number and CI.
2. **Reproduce the environment.** Same harness version, same transformers / vllm version, same tokenizer, same dataset revision, same seed. Log all of them in a `MANIFEST.md`.
3. **Run it.** Deterministic (`--seed 1234`, `--num_fewshot N`, matching batch size if the harness is batch-size-sensitive), with `--log_samples`.
4. **Compare aggregates.** Point estimate and bootstrap SE. If within tolerance, you are done; write it up.
5. **If off, compare per-item.** Load the reference per-item log (if published) or the reference sample count and re-derive the aggregate to check the arithmetic. Look for systematic per-item differences (are the items where you differ mostly of one type?) and use them to guess which of the four gap sources is in play.
6. **Write the gap-analysis note.** One page: the target, the reproduction attempt, the delta, the diagnosed cause, and the corrective config that would close the gap. If the gap cannot be closed, note the residual and its most-likely explanation. This note is the artifact that makes your reproduction cite-able.

## A note on HELM vs. lm-eval numbers for the "same" task

HELM's `mmlu` and lm-eval's `mmlu` will not produce the same number for the same model, and this is by design.

- HELM's default MMLU adapter is `multiple_choice_joint` (generation), 5-shot, with a specific instruction prefix and a specific letter-extraction rule.
- lm-eval's default MMLU task uses `multiple_choice` (log-likelihood ranking with `acc`), 5-shot, with a different prompt template.

The two numbers on the same model on the same underlying dataset can differ by 5+ points, and neither is wrong. When someone asks "what's the model's MMLU," the correct response is "under which harness?" This is why every serious internal launch report specifies the harness, version, and adapter, not just the dataset name.

## Summary

HELM's `Scenario` / `Adapter` / `Metric` / `RunSpec` decomposition is the cleanest published articulation of the four-objects taxonomy, and its portable-scenario-plus-composable-metrics design supports the matrix-style reports HELM is built around. Register a scenario, bind it to an adapter, attach metrics, and run with `helm-run`. Reproducing a public number is a four-step workflow: figure out what was run, reproduce the environment, run it, and write a gap-analysis note. When the rerun does not match, the four gap sources — prompt template, decoding, normalisation, dataset revision — cover almost every real case; diagnose in that order. "Within tolerance" means inside the published CI, not identical to five decimal places. Two harnesses' "same" task will not agree, and this is not a reproducibility failure — it is why every published number is qualified by its harness. Chapter 5 turns to Inspect, whose model of an eval is different again and better suited to the multi-turn safety and agent evals lm-eval and HELM struggle with.
