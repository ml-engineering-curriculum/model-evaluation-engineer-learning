# exercise-04: OpenAI-Evals-Style Registry Task with Model-Graded Scoring

**Estimated effort:** 4 hours

## Objective

Write a full [OpenAI evals](https://github.com/openai/evals)-style registry entry — YAML eval spec, JSONL samples file, and a `modelgraded` rubric — for an open-ended generation task where exact match is not applicable. Run the eval with two grader models (one small, one large). Then do the thing every published model-graded score needs and almost none has: **validate the grader against a human-labelled subset** and report Cohen's κ between the grader and the humans. Your deliverable is a working eval whose grader has a measured trust level, not just a plausible-looking number.

## Prerequisites

- mod-104 Chapters 1, 6 (harness taxonomy, OpenAI evals registry + model-graded).
- mod-102 Chapter 4 (inter-annotator agreement, adjudication).
- The `evals` repo cloned and installed (`pip install evals` or the [dev install](https://github.com/openai/evals#setup)).
- API credentials (OpenAI or an OpenAI-compatible endpoint) for at least two models to serve as graders. Reasonable pairs: `gpt-4o` + `gpt-4o-mini`, `gpt-4o-mini` + `gpt-3.5-turbo`, or a hosted commercial grader + a self-hosted open-weights grader via `openai`-compatible endpoint.

## The task: pick an open-ended one

The task must be one where exact-match scoring is not sensible. Suggested shapes:

- **Explanation quality.** "Explain <topic> to a 12-year-old in ≤ 100 words." Dataset seed: 30–100 topics drawn from Wikipedia / a school curriculum.
- **Summarization faithfulness.** "Summarize this article in one paragraph." Dataset seed: XSum, CNN/DailyMail, or your own article set. Rubric: faithfulness (does the summary contain unsupported claims?).
- **Refusal / safety compliance.** "Given this request, either comply with a safe answer or refuse politely." Dataset seed: a mix of clearly-safe requests and clearly-unsafe requests. Rubric: correct refusal / correct compliance.
- **Task instruction following.** "Given a set of formatting constraints, produce output following them." Dataset seed: a subset of IFEval-style tasks. Rubric: constraint satisfaction.
- **Domain-specific rubric.** Your own domain (medical Q&A, legal summarization, code review comments) — as long as the labels are legitimately shareable and the rubric is defensible.

Constraints:
- **Dataset size:** 50–200 items for the full eval. 30–50 items for the grader-validation subset (subset of the full set).
- **The task must have real ambiguity in the middle** — if every response is trivially "correct" or "incorrect," you don't need model-graded scoring and the exercise is uninteresting.
- **The reference answer must be write-down-able** — a rubric that requires the grader to *judge from scratch* rather than *compare to a reference* is much less reliable (Chapter 6). If your task genuinely has no reference answer (open creative generation), the rubric must be constrained to criteria the grader can evaluate independently (length, format, safety) rather than "was it good."

## Requirements

### Part A — dataset (`data/`)

Produce `data/<taskname>/samples.jsonl`, one JSON object per line, following the OpenAI evals conventions:

- `input`: a list of chat messages (`role` + `content`) that the subject model will see.
- `ideal`: the reference answer (string or list of acceptable strings). For rubric evals where "correctness" is a comparison to a reference, this is what the rubric will substitute in.
- `id`: a stable per-item identifier.
- Optional: any metadata you want available to the rubric (category, difficulty, region).

If your task has a reference answer but is open-ended in phrasing (a rubric will judge equivalence rather than exact string), include the reference in `ideal` and design the rubric to compare against it (Chapter 6's rule).

### Part B — the registry entry (`evals/`)

Write `my_registry/evals/<taskname>.yaml` with two YAML documents:

1. The top group with `id`, `description`, and `metrics`.
2. The versioned eval spec binding either `evals.elsuite.basic.match:Match` (as a baseline floor) *or* `evals.elsuite.modelgraded.classify:ModelBasedClassify` (for the graded variant). The graded variant is the required one; the floor is a stretch.

For the graded variant, include:

```yaml
<taskname>.mg.v0:
  class: evals.elsuite.modelgraded.classify:ModelBasedClassify
  args:
    samples_jsonl: <taskname>/samples.jsonl
    eval_type: cot_classify
    modelgraded_spec: <rubric_name>
```

### Part C — the rubric (`modelgraded/`)

Write `my_registry/modelgraded/<rubric_name>.yaml`. The rubric must follow Chapter 6's rules:

- **Bounded categorical output** — 2–4 choices, each with a single-letter label.
- **Reference included in the prompt** — the rubric compares subject output to `{ideal}`, does not judge from scratch.
- **Chain-of-thought before the verdict** — use `eval_type: cot_classify` (or write the "reason first, letter second" pattern into the rubric explicitly).
- **`choice_scores`** mapping categorical output to a numeric score.
- **`choice_strings`** enumerating the valid labels.
- **`input_outputs`** mapping the eval's variables into the rubric's slots.

Rubric shape (adapt to your task):

```yaml
prompt: |
  You are an expert grader for a <task description> task. You will compare
  a model's answer to a correct reference and decide how well it matches.

  Task:
  """
  {input}
  """

  Correct reference:
  """
  {ideal}
  """

  Model's answer:
  """
  {completion}
  """

  First, reason briefly about whether the model's answer captures the same
  content as the reference. Then respond with a single letter:
  A) Fully correct — captures all the key points of the reference.
  B) Partially correct — captures some key points, misses others, or adds
     unsupported claims.
  C) Incorrect — misses the main content or contradicts the reference.

  Reasoning:

choice_scores:
  A: 1.0
  B: 0.5
  C: 0.0

choice_strings: ["A", "B", "C"]

input_outputs:
  input: completion
```

### Part D — run the eval against two graders

Run the eval with each grader:

```bash
oaieval <subject_model> <taskname>.mg.v0 \
  --registry_path ./my_registry \
  --modelgraded_completion_fn <grader_model_1> \
  --record_path logs/<taskname>_g1.jsonl

oaieval <subject_model> <taskname>.mg.v0 \
  --registry_path ./my_registry \
  --modelgraded_completion_fn <grader_model_2> \
  --record_path logs/<taskname>_g2.jsonl
```

The subject model can be one and the same across the two runs; the changes are on the grader. Report per-grader:
- Aggregate score (mean over items, with a bootstrap 95% CI).
- Cost / latency of grader calls.

### Part E — grader validation against humans (the key part)

Sample 30–50 items from the full eval as a **validation subset**. Have **at least 2 humans** (you + at least one other; even a friend or teammate is fine) label each item independently on the same rubric the grader uses (`A` / `B` / `C` or whatever your labels are). If they disagree, adjudicate to a single label per item (mod-102 Chapter 4).

Then compute, for each grader:

- **Cohen's κ** between the grader's per-item label and the adjudicated human label. (`sklearn.metrics.cohen_kappa_score`.)
- **Accuracy** (fraction of items where the grader's label matches the human label).
- **Per-label confusion matrix** — where does the grader systematically disagree with humans?

Report all three per grader. Discuss:

- Which grader is closer to human judgment?
- Are the graders systematically biased in a specific direction (e.g. more lenient than humans, favoring longer responses)?
- Given the κ values, is the graded eval usable for high-stakes decisions? (Chapter 6: `κ ≥ 0.7` usable; `0.5 ≤ κ < 0.7` usable with caveats; `κ < 0.5` needs rubric revision.)

### Part F — the report

Write `REPORT.md` (≤ 2 pages) with:

1. **Task description.** What the eval measures, why exact match is not sufficient.
2. **Rubric design decisions.** Why the specific choice set, why the specific reference-comparison framing.
3. **Grader results table** — per grader, mean score with 95% CI, cost, latency.
4. **Grader-vs-human agreement table** — per grader, κ and accuracy against humans, with the confusion matrix.
5. **Grader selection.** Which grader to ship as the "canonical" grader for this task, and why (agreement × cost trade-off).
6. **Limitations.** What kinds of items are the graders most likely to be wrong on? What is the smallest human-validation set that would let you monitor for grader drift over time?

### Part G — reproducibility

Ship:
- `data/<taskname>/samples.jsonl`
- `my_registry/evals/<taskname>.yaml`
- `my_registry/modelgraded/<rubric_name>.yaml`
- `human_labels.csv` — the human labels used for grader validation, with per-annotator columns and the adjudicated column.
- `logs/*.jsonl` — the full grader run logs.
- `MANIFEST.md` — versions, models, seeds.
- `REPORT.md`
- `run.sh` — one-shot rerun of both grader runs.

The human labels file is a sensitive artifact — do not include real personal or unsafe content in it. If your task's items or human labels are sensitive, publish only the aggregate agreement numbers and note the omission.

## Starter guidance

- **Pilot the rubric on 5 hand-picked items before running the full eval.** Include 2 clearly-correct, 2 clearly-wrong, and 1 borderline. If the grader gets the 4 clear cases right and its reasoning on the borderline case is plausible, the rubric is worth running at scale. If it gets a clear case wrong, revise the rubric before you spend money.
- **Do not use the same grader model as the subject model.** A model grading its own outputs has strong self-preference biases (see Panickssery et al. 2024 in `resources.md`) and gives inflated scores.
- **Chain-of-thought before the verdict is worth the tokens.** `eval_type: cot_classify` in OpenAI evals prompts the grader to reason first; on non-trivial rubrics this typically moves grader-human κ by 5–15 points.
- **Cohen's κ is prevalence-sensitive.** If most of your items are "A / fully correct," a grader that always says A has high accuracy but low κ. Report both. If the label distribution is very skewed, use Krippendorff's α as a cross-check.
- **Human labelling takes longer than you expect.** Budget 30–60 seconds per item per rater. 30 items × 2 raters ≈ an hour of human time. Do this before running the graders so you are not tempted to look at the grader's answer before making your own.
- **Include the adjudication process in `human_labels.csv`.** Which items the two humans disagreed on, and how you resolved. A validation set with unaddressed inter-rater disagreement is weaker than a smaller set with adjudicated labels.
- **The Chapter 6 rubric-design constraints are load-bearing.** Skipping "include the reference in the prompt" or "constrain to a categorical output" is the most common way rubrics fail. If your grader is getting low κ, revisit the rubric against those rules before revisiting the grader model.

## Acceptance criteria

- The eval runs end-to-end via `oaieval` (or an equivalent runner if you are on a compatible endpoint) with a single command captured in `run.sh`.
- The rubric YAML follows Chapter 6's four rules (bounded output, reference in prompt, few coarse categories, CoT-before-verdict).
- The eval is run with two grader models and per-grader aggregate scores are reported with 95% CIs.
- A grader-validation subset of 30–50 items has been human-labelled by ≥ 2 annotators with an adjudication protocol.
- Per-grader Cohen's κ against adjudicated human labels is reported alongside the aggregate score. A grader score without a κ is not acceptable.
- The `REPORT.md` makes an explicit grader-selection call and justifies it against the agreement / cost trade-off.
- The `human_labels.csv` is included in the deliverables and matches the κ computation.
- The bundle is self-contained: a reviewer with the API credentials could rerun both grader passes and reproduce the aggregate.

## Stretch goals

- **Self-preference stress test.** Add a third grader — the *same* model as the subject — and quantify the self-preference bias by comparing its verdict against the two other graders and against humans. The delta is worth publishing.
- **Rubric variant sensitivity.** Write two additional rubric variants (a stricter and a more lenient wording) and rerun the graded eval under each. Report the per-grader × per-rubric matrix. Rubric variance is often the largest source of noise in model-graded evals; measuring it makes the number's error bars honest.
- **Grader drift monitoring.** Simulate a "future" grader release by rerunning the eval with a different grader-model version (e.g. `gpt-4o` at a different `--completion_args`) and quantify the delta from grader upgrade alone. Sketch how you would monitor this in production.
- **Cost-aware sub-sampling.** For a large eval, propose a stratified sub-sample that gives you 90% of the κ signal at 30% of the grader cost. Justify the sample design and rerun to verify.
- **Human-validation set turnover.** Discuss how you would rotate the human-validation set every N months to avoid the grader over-fitting to the specific items (a real concern if the rubric is tuned against validation-set κ).
