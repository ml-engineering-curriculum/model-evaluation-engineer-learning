# exercise-02: Reproduce a Public HELM or lm-eval-harness Result

**Estimated effort:** 4 hours

## Objective

Pick a *specific published benchmark number* — a leaderboard row or a model-card claim — and reproduce it end to end. Your deliverable is a reproduction attempt with a documented gap-analysis: either you land inside the reported CI and can write a MANIFEST that reproduces the number, or you land outside it and can name which of the four gap sources (Chapter 4) explains the delta. Both outcomes are valid completions. The point of the exercise is that a published LLM number is a *claim you can test*, and this exercise is the workflow for testing one.

## Prerequisites

- mod-104 Chapters 1–4 (harness anatomy, lm-eval, log-likelihood-vs-generation, HELM + reproduction workflow).
- `lm-eval` v0.4.x installed *or* `crfm-helm` installed (only one is required — the reproduction target dictates which).
- An open-weights model checkpoint downloadable from HF Hub, or API credentials for a hosted model whose published benchmark number you are chasing. Choose a model you can actually run — reproducing a GPT-4-class API number costs real money and is a valid but expensive choice.

## The target: pick a specific number

Pick **one** row from **one** published source. The row is a `(benchmark, task, model, harness, harness_version)` tuple; every field must be pin-down-able. Reasonable target sources:

- The [Hugging Face Open LLM Leaderboard v1 archive](https://huggingface.co/spaces/open-llm-leaderboard-old/open_llm_leaderboard) — MMLU / HellaSwag / ARC / TruthfulQA / GSM8K / Winogrande on open-weights models. Uses lm-eval.
- The [Open LLM Leaderboard v2 methodology note](https://huggingface.co/docs/leaderboards/open_llm_leaderboard/about) — MMLU-Pro / GPQA / MuSR / BBH / IFEval / Math-hard on open-weights models. Uses lm-eval; different config from v1.
- The [HELM classic leaderboard](https://crfm.stanford.edu/helm/) or a HELM sub-leaderboard (HELM Lite, HELM MMLU, HELM Instruct) — many scenarios across many models. Uses HELM.
- A **model card** claim (Meta Llama-3, Mistral, Qwen2, Gemma-2, DeepSeek). Model cards usually cite lm-eval or HELM and give a config; you may need to dig for the exact task version.
- An **arxiv paper** reporting an lm-eval / HELM number with a citable methodology section.

Constraints on the target:
- **The harness must be one of the four covered in this module.** Skip papers that ship a bespoke script — reproducibility of those is a research project, not a 4-hour exercise.
- **The task must be one you can register or reuse.** Prefer targets where the task is in the harness's default registry.
- **The model must be runnable in your environment.** No 70B checkpoints on a single 24GB card. If in doubt, prefer 7–8B open-weights checkpoints.
- **Prefer tasks with published bootstrap SEs / CIs.** "Within tolerance" is meaningful only when there is a tolerance to compare against.

Reasonable difficulty-graded picks:

- **Easy:** HellaSwag `acc_norm` for an open-weights 7B on Open LLM Leaderboard v1 (lm-eval, log-likelihood). Deterministic, well-documented task, few surprises.
- **Medium:** MMLU `acc` on a chat-tuned model. Sensitive to chat-template application; the gap analysis is the interesting part.
- **Hard:** GSM8K `exact_match` on a chat-tuned model. Sensitive to `max_gen_toks`, filter regex, decoding config, and chain-of-thought variant. Reproducing to within CI often requires all four to be pinned right.
- **Hard (HELM):** HELM classic MMLU on an open-weights model — reproducing the HELM number specifically (not the lm-eval number of the same name) requires using HELM's `multiple_choice_joint` adapter with the exact HELM prompt.

Write down the target in a `TARGET.md` file: benchmark, task, subtask if any, model, harness + version, published value, published CI, and the URL of the source.

## Requirements

### Part A — TARGET.md (the claim you are testing)

Before running anything, fill in a `TARGET.md`:

```
Benchmark:       <e.g., Open LLM Leaderboard v1 - MMLU>
Task:            <e.g., mmlu (5-shot, average across 57 subjects)>
Model:           <HF repo + revision, or API model + version>
Harness:         <lm-eval v0.4.2 / crfm-helm v0.5.4 / other>
Adapter/evaluator: <log-likelihood-ranking with acc / joint MC with exact_match / ...>
Published value: <point + CI or SE>
Published as:    <URL of the leaderboard / model card / paper>
Configuration:   <known num_fewshot, decoding, chat_template applied, etc.>
```

The `Configuration` line is the most important. If a field is not documented in the source, mark it `unknown — will use harness default` and note this. Every unknown is a potential gap source.

### Part B — the reproduction environment

Set up an environment that pins:

- Harness version (git commit, not just a version tag if possible).
- Model checkpoint (HF revision hash).
- Dataset revision (HF revision hash) for the underlying task.
- Library versions (`transformers`, `tokenizers`, `torch`, `vllm` if used).

Record all of this in a `MANIFEST.md` — see Chapter 7 for the field list. If a pin is not achievable (the source did not publish the harness commit and the version tag has moved), document the gap.

### Part C — reproduce

Run the eval end-to-end. Use `--log_samples` (or the equivalent) so you can inspect per-item results. The run command belongs in `run.sh` and its exact form in `MANIFEST.md`.

- For lm-eval, mirror the published `--num_fewshot`, `--apply_chat_template`, `--fewshot_as_multiturn` flags, and decoding config.
- For HELM, use the same `--run-specs` string the leaderboard uses (published in the HELM sub-leaderboard's methodology).
- Log the *rendered* prompt for at least the first item and diff it against a rendered prompt from the published run if one is available (HELM viewer exposes per-instance prompts; lm-eval's samples JSONL does the same).

### Part D — compare

Produce a table:

| item | published | your run | delta |
|---|---|---|---|
| point estimate | ... | ... | ... |
| standard error / CI half-width | ... | ... | ... |
| n_test | ... | ... | ... |

Classify the outcome:

- **Reproduced:** your point estimate is inside the published CI, or you and the published number agree to within ±(your SE + published SE).
- **Not reproduced:** otherwise.

There is no partial credit for "close enough" without the CI comparison.

### Part E — the gap-analysis report

Whether or not you reproduced, write `REPORT.md` (≤ 2 pages) with:

1. **Summary** — target, reproduction outcome, delta.
2. **Reproduction environment** — a condensed MANIFEST, one paragraph.
3. **Gap analysis** (the interesting part), structured by the four sources from Chapter 4:
   - **Prompt / chat template & few-shot.** Did the rendered prompt match the published one, character for character? For chat-tuned models, was the chat template applied? Was the few-shot selection deterministic?
   - **Decoding config.** For generation-mode tasks: `do_sample`, `temperature`, `max_gen_toks`, stop sequences. Cite the source's value alongside yours.
   - **Normalisation / metric flavour.** `acc` vs. `acc_norm`? `exact_match` vs. `quasi_exact_match`? Which one is on the leaderboard? Which one did you compute?
   - **Dataset revision.** Same HF revision as the published run? If not, could you rerun with the historical revision?
4. **What closed (or would close) the gap.** If you reproduced, note the specific alignment that got you there (usually: chat template application, or matching the exact few-shot config, or picking the right normalisation). If you did not reproduce, name the single most-likely explanation and the experiment that would confirm it (which you are not obligated to run in this exercise).
5. **A short reflection.** One paragraph on which of the four gap sources dominates for the class of task you picked. Log-likelihood tasks tend to be prompt-template-dominated; generation tasks tend to be decoding-dominated; math/code generation tasks tend to be filter-dominated.

### Part F — the reproducibility handoff

The final submission is:
- `TARGET.md`
- `MANIFEST.md`
- `run.sh` (exact commands, one-shot rerunnable)
- `results/` directory with `results.json` (or HELM equivalent) and per-sample logs
- `REPORT.md`
- Optional: a diff of the rendered prompt (yours vs. the published) if you can obtain the published prompt.

A reviewer with the same hardware should be able to run `bash run.sh` and get numbers matching your `results/` outputs to floating-point noise.

## Starter guidance

- **Pick the smallest defensible target that has a real CI.** Do not chase a headline MMLU aggregate if a single subject task will let you exercise the workflow — but if you pick a single subject, be transparent that you did not reproduce the aggregate.
- **The published CI is usually a bootstrap SE, not a Wilson interval.** Compare like with like: use the harness's bootstrap SE for your rerun's CI unless you have a reason to compute Wilson from scratch.
- **HF datasets rev drift is real.** If you hit an unexpected delta of ~0.5 points and the config matches, check whether the dataset's HF revision changed since the published run. Not all published runs record the dataset revision — this is often the "unknown" that causes the gap.
- **On chat-tuned models, always check whether the published run used the chat template.** Model cards commonly *do*; the Open LLM Leaderboard v1 did *not* until later revisions. This is worth 5–15 accuracy points and is the most common gap source in practice.
- **If your rerun is way off (5+ points), stop and rule out the "wrong model checkpoint" case first.** Confirm the revision hash. A quantized checkpoint served under a non-quantized name is a real occurrence.
- **HELM reproduction is more work than lm-eval reproduction** because HELM has richer per-scenario adapters and normalisations. Budget accordingly. If in doubt, do lm-eval this round and pick up HELM in the stretch goals.
- **The exercise is not "hit the number." It is "explain the number, either way."** A precisely diagnosed non-reproduction is a stronger deliverable than a not-quite-honest reproduction.

## Acceptance criteria

- `TARGET.md` specifies a testable claim: benchmark, task, model, harness, version, published value with CI, source URL.
- `MANIFEST.md` contains every pin from the Chapter 7 checklist. Unknowns are marked, not guessed.
- `run.sh` reruns the eval end-to-end and produces `results.json`.
- The comparison table in `REPORT.md` includes both point estimate and CI half-width for both the published and reproduced number.
- The outcome is classified as reproduced or not, using the CI comparison.
- The gap analysis walks all four Chapter-4 sources; for each, either the source is ruled out with evidence or it is diagnosed as (partially) responsible.
- If you did not reproduce, the report names a single most-likely mechanism and the experiment that would confirm it. If you did reproduce, the report names the specific alignment that closed the gap.
- The rendered prompt for at least one item is inspectable in the deliverables.

## Stretch goals

- **Two harness reproductions of the same task.** Reproduce MMLU on the same open-weights model under both lm-eval and HELM. Confirm the two numbers are *not* the same and quantify the harness-driven delta. Write a paragraph on which harness's number is "more comparable" to which downstream question.
- **Seed variability.** For a generation-mode task with sampling, run 8 seeds and report the mean and 95% CI across seeds. This gives you an empirical read on the seed-driven noise floor for that task, which many published numbers gloss over.
- **Historical-revision rerun.** If your dataset has multiple HF revisions (MMLU does), rerun against the specific revision the published number used and against the current revision, and quantify the delta attributable to dataset drift alone.
- **Blind reproduction.** Give the TARGET to a peer and have them attempt the reproduction without seeing your MANIFEST. Then compare notes on what they had to guess. This is the fastest way to see which pins are load-bearing in practice.
