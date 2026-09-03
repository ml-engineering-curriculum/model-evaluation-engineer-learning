# exercise-05: Prompt-Format Sensitivity Diagnostic

**Estimated effort:** 4 hours

## Objective

Take a single LLM benchmark task and produce a **sensitivity report** that separates task signal from harness-plumbing noise. You will sweep prompt template, choice-order permutation, few-shot demonstration seed, decoding config, and normalisation on the same task and the same model(s), then report the score as a distribution across the sweep — median, spread, and rank-stability — rather than as a single point. The deliverable is a report that lets a reader answer: *"How much of the headline number is the model, and how much is the prompt?"* This is the discipline Chapter 7 argued every serious eval report needs and almost none has.

## Prerequisites

- mod-104 Chapters 1–7 (all chapters; Chapter 7 is the direct spec for this exercise).
- mod-101 Chapters 3–4 (Wilson CIs and bootstrap CIs).
- mod-102 Chapter 3 (dataset revision and hashing).
- A working install of at least one harness (`lm-eval` is the recommended default; Inspect or OpenAI evals work if that is where the target task lives).
- A model you can run reproducibly (any small open-weights model runnable locally or via a served OpenAI-compatible endpoint). Two models are strongly encouraged for the rank-stability check.

## The task: pick one

Pick a **single, well-known task** that is documented, small, and stable enough that you can run 30+ variants without exhausting your compute or wall-clock budget. The task must be one you can register or reuse in your chosen harness.

Reasonable picks:

- **A single MMLU subject** (e.g. `mmlu_high_school_biology`, `mmlu_college_computer_science`). ~150 items per subject; log-likelihood-friendly; every published number carries a specific prompt template.
- **HellaSwag** (a random 500-item subset). Longer candidates make the length-normalisation axis interesting.
- **ARC-Easy** or **ARC-Challenge** (a 500-item subset). Short candidates; the choice-order permutation axis is where the drama is.
- **BoolQ** (a 500-item subset). Two choices; prompt-template variance dominates.
- **GSM8K** (a 200-item subset). Generation-mode; `max_gen_toks` and CoT-prompt variance dominate.
- **A single-scenario HELM run** or **an Inspect eval** you built in exercise-03. Fine as long as you can vary the axes below.

Cap the item count to what lets you run **at least 20 variants** within budget. Sensitivity is a distribution; a diagnostic on 3 variants is not a diagnostic.

## Axes to vary

You do **not** need the full factorial. A defensible design is one-at-a-time (OAT) sweeps from a chosen **baseline configuration**, with a small factorial (~4 cells) over two axes chosen for their expected interaction.

Vary at minimum three of these six axes; the first two are required.

1. **Prompt template (required).** 5–8 semantically-equivalent variants of the template. Examples for MC: `"Q: {q}\nA:"`, `"Question: {q}\nAnswer:"`, `"### Question\n{q}\n\n### Answer:"`, `"[INST] {q} [/INST]"`, `"{q}\nCorrect answer:"`, `"{q}\n\nThe correct answer is"`. Choice-letter format: `"A. {choice}"` vs. `"(A) {choice}"` vs. `"[A] {choice}"` vs. `"A) {choice}"`. Pick a set that covers the space, not variations of a single style.
2. **Choice-order permutation (required for MC).** For an MC task, permute the order of the choices at least 3 ways: canonical order, reversed order, and one shuffled seed. If your task allows, run all `k!` permutations for small `k` and report position-conditional accuracy.
3. **Few-shot demonstration seed.** 3–5 different values of `--seed` (or `fewshot_seed`) that produce different demonstration selections. If the harness lets you fix the demonstration IDs, hand-pick a "hard-demo" and "easy-demo" set as extreme cases.
4. **Chat template on/off.** For a chat-tuned model, run once with `--apply_chat_template --fewshot_as_multiturn` and once without. This is Chapter 2's single-most-common-bug axis; measuring the delta is the point.
5. **Decoding config (generation tasks only).** Greedy (`T=0`) vs. sampled (`T=0.7`, `top_p=0.95`, 5 seeds). For generation-mode tasks, also vary `max_gen_toks` at two settings (a "too short" value that truncates CoT and a "generous" value).
6. **Normalisation (log-likelihood MC tasks only).** `acc` vs. `acc_norm` vs. a PMI-normalised score (requires a second log-likelihood call per candidate with a null / task-generic context).

Log every cell in the sweep in a `SWEEP.md` design document before running.

## Requirements

### Part A — the baseline and MANIFEST

Pick a **baseline configuration** (the canonical prompt for your task, the harness default decoding, the canonical `num_fewshot`, the canonical normalisation). This is the reference point every OAT sweep varies from.

Write `MANIFEST.md` per Chapter 7 for the baseline: harness version + commit, model + revision, dataset + revision, decoding, seed, few-shot config, prompt-template hash, environment, hardware. Every variant in the sweep varies exactly one of these fields; log the delta from baseline for each cell.

### Part B — register the variants

For every cell in the sweep, produce a runnable configuration. Depending on your harness:

- **lm-eval.** One YAML per template variant under `tasks/<taskname>_v01.yaml` ... `tasks/<taskname>_v08.yaml`. Non-template axes (few-shot seed, chat template, normalisation) become CLI flags. A group file lets you run all template variants in one command.
- **Inspect.** Parameterize the solver on the template as an argument (`prompt_template(template=<variant>)`) and use `--task-arg` to sweep. Choice-order permutation lives in the dataset loader.
- **HELM.** Each variant is a separate `RunSpec`; register one per template variant.
- **OpenAI evals.** Each template variant is a different `samples.jsonl` (rendered with the target template). Decoding lives in `--completion_args`.

### Part C — run the sweep

Run every cell against **at least one model**, and preferably **two models of different families** (for the rank-stability check). All runs must share the same harness version, dataset revision, and model revision — only the axes above vary.

Log every run under `runs/<axis>=<value>/` with the full per-sample log (`--log_samples` or the harness equivalent). Never trust a sweep whose per-sample logs you cannot inspect.

Budget hint: if a single run takes N minutes, a 20-cell sweep against 2 models is 40N minutes of GPU time; plan accordingly. Use `--limit` liberally on early runs to catch config bugs before you commit compute.

### Part D — aggregate as a distribution

Produce `results/summary.csv` with one row per (model, cell) and columns: axis, value, metric name, point estimate, bootstrap SE, n_items. Then build:

1. **Per-axis distribution table.**

   | model | axis | n_variants | median | 5th | 95th | min variant | max variant |
   |---|---|---|---|---|---|---|---|
   | llama-3-8b-inst | template | 8 | 0.612 | 0.554 | 0.658 | `[INST]` | `### Question` |
   | llama-3-8b-inst | choice-order | 4 | 0.615 | 0.591 | 0.638 | reversed | canonical |
   | ...

   The 5th–95th percentile span across variants is the honest error bar on this task-model pair. If the span is wider than the difference between models you care about, the model comparison is not robust.

2. **Rank-stability check (two-model runs only).**

   | axis | fraction of variants where model_A > model_B |
   |---|---|
   | template | 6 / 8 |
   | choice-order | 4 / 4 |
   | fewshot-seed | 5 / 5 |
   | ... | |

   If the fraction is `1.0` (or `0.0`), model_A dominates (or is dominated) across the axis. Anywhere between `0.4` and `0.6` and the pair-wise comparison is not robust to that axis.

3. **Per-item drift.** For the two extreme cells on the largest-effect axis (highest and lowest median score), join the per-sample logs on `doc_id` and report: fraction of items where the prediction flipped, fraction of items that the model got right in *both* cells (the "task-signal" subset), fraction wrong in both cells (the "task-noise" subset), and the two flip-direction cells. This is where you separate signal from plumbing at the per-item level.

4. **Bootstrap CIs on the median-across-variants.** For each (model, axis), bootstrap the median across variants (resample cells with replacement) to get a CI on the median itself. This CI is what a reader should compare across models, not the CI of any single-variant point estimate.

### Part E — the sensitivity report

Write `REPORT.md` (≤ 3 pages) containing:

1. **Task and baseline.** Task name, harness + version, model(s), baseline configuration, and headline number *if a reader forced you to pick a single one*.
2. **Sweep design.** Which axes, how many cells per axis, why those choices (why 8 templates rather than 4, why these specific letter formats).
3. **Per-axis distributions.** The table from Part D.1. One paragraph per axis on what the spread means (e.g. "template spread is 10 points on this model; the model's headline number sits at the 40th percentile of the template distribution — this is the ceiling on how tightly this task can measure the model").
4. **Rank-stability.** For two-model runs, the table from Part D.2 and a two-sentence takeaway on whether the model comparison is robust.
5. **Signal vs. noise.** For the largest-effect axis, the per-item drift table from Part D.3 and 3–5 example items where the prediction flipped between the two extreme cells. What kind of items flip? Are they concentrated in a subset (long-answer items, items with unusual choice length, items where the model was already borderline)?
6. **Recommendation.** Given the sensitivity you measured, what is the *correct* headline number to report for this task on this model? Options: median-across-variants with the 5–95th CI (the most honest); baseline-config point estimate with a note on template sensitivity; report as a matrix. Pick one and defend the choice.
7. **What this means for the published number.** If the task has a published leaderboard number, comment on whether that number sits inside your variant distribution. If it does, the published number is defensible; if it does not, either your sweep missed the published cell or the published cell is an outlier — say which.

### Part F — reproducibility

Ship:
- `SWEEP.md` — the sweep design.
- `MANIFEST.md` — the pins for the baseline.
- `tasks/` or `variants/` — every registered variant.
- `run.sh` — a single script that runs the entire sweep end-to-end and reproduces `results/summary.csv`.
- `results/` — per-cell results JSONs, per-sample logs, and `summary.csv`.
- `REPORT.md`.

A reviewer with the same hardware and pinned versions should be able to run `bash run.sh` and reproduce every point estimate to within floating-point noise.

## Starter guidance

- **Design the sweep on paper first.** A sensitivity report is only useful if the axes actually correspond to axes readers care about; a sweep over 8 templates that all look like `"Q:/A:"` variants is not measuring what you think.
- **Do the OAT sweep before the factorial.** Full-factorial designs are almost never worth the compute for prompt-sensitivity work; a per-axis distribution reveals almost everything, and a small factorial (say template × chat-template) can pick up the one or two interactions that matter.
- **Pin every non-swept knob.** The sweep only measures the axes you vary. If your few-shot seed drifts between cells because you forgot to fix it, you have measured a mixture of axes and the per-axis attribution is invalid.
- **Use `--limit N` on early runs.** A 20-cell sweep with a config bug in cell #1 that only surfaces at item #240 will waste hours. Run every cell on `--limit 20` first, sanity-check the rendered prompt for each, then run the full sweep.
- **The largest per-axis spread is not always the largest theoretical concern.** Template spread of 10 points looks scary; but if all your candidate models sit at the *same relative position within the spread*, model comparisons are still robust. Rank-stability is the load-bearing metric for decision-making, not raw spread.
- **On chat-tuned models, always include the `--apply_chat_template` on/off comparison.** It is the highest-EV single comparison in the sweep and usually the largest single-axis delta.
- **Bootstrap the median, not the mean.** Sweeps often have a long tail (one broken template, one aggressive normalisation) that a mean absorbs and a median rejects. Report the median with a bootstrap CI on the median.
- **Per-item drift analysis is where the qualitative insight lives.** Two variants can have the same accuracy but disagree on 40% of items — that is a very different task-signal claim than two variants with 3% item disagreement. Compute the disagreement, do not infer it.

## Acceptance criteria

- `SWEEP.md` documents the axes, cells, and baseline before any run is executed.
- `MANIFEST.md` pins the baseline per Chapter 7's checklist.
- At least 3 axes are swept; template and choice-order (for MC) or template and decoding (for generation) are among them.
- At least 20 total (axis, value) cells were run per model.
- `results/summary.csv` has one row per (model, cell) with point estimate, SE, and n_items; no bare medians without their spread.
- The per-axis distribution table in `REPORT.md` includes 5th, 50th, and 95th percentiles and names the variant at each extreme.
- If two models were run, the rank-stability table is present and interpreted.
- The per-item drift analysis identifies flipped items and categorizes at least 3 of them qualitatively.
- The report picks one reporting mode (median-with-CI vs. baseline-with-note vs. matrix) and defends the choice.
- `run.sh` reproduces every cell in the sweep from the pinned MANIFEST.

## Stretch goals

- **Two-model rank flip.** If your two-model sweep shows the models trading rank across variants, find the minimum-sized subset of the sweep on which the rank *always* flips — this is the sharpest possible demonstration that the pair-wise comparison is prompt-sensitive rather than model-sensitive.
- **Harness cross-check.** Register the same task in two harnesses (e.g. lm-eval and HELM). Run the baseline configuration under both. Report the harness-driven delta as an *additional* axis on the sensitivity report. Chapter 4 warned this delta can dwarf the within-harness template spread; measure it.
- **Sample-efficient sweep design.** Given a budget of N total inference calls, propose the OAT axis-and-cell allocation that maximizes information about the sensitivity distribution. Compare against uniform allocation.
- **Sensitivity under adversarial re-templating.** Add a template variant explicitly designed to *hurt* the model (missing punctuation, ambiguous instruction). This is a rough lower-bound on the "hostile-prompter" scenario the task might see in production.
- **Publish the sweep as a Model Card row.** Draft the paragraph you would include in a model card that reports the median-across-variants number and its spread instead of a bare point estimate. This is the artifact that turns the exercise from "internal diagnostic" into "external claim."
