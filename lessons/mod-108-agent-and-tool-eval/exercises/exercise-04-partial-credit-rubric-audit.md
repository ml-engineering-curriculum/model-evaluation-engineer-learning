# exercise-04: Partial-Credit Rubric Audit

**Estimated effort:** 3 hours

## Objective

Design a partial-credit rubric for a set of free-form agent trajectories, run it as an LLM-as-judge, and **audit the judge against a human-labelled gold slice** so the rubric's numbers can be defended. The deliverable is a rubric spec, a small human-labelled gold set, a judge-vs-human agreement report, and a `use / re-anchor / do-not-use` decision per rubric dimension. Chapter 5 is the design frame; mod-105's calibration protocol is the audit mechanic.

The exercise is deliberately not tied to one benchmark. You choose the trajectory source; the discipline is the same regardless.

## Prerequisites

- mod-108 Chapter 5 (partial-credit rubrics for trajectories).
- mod-105 Chapter 5 (calibrating a judge against humans) and Chapter 4 (bias controls).
- mod-102 Chapter 4 (inter-annotator agreement — you will compute κ).
- 40–80 agent trajectories in a readable format. Source options: WebArena/GAIA traces from exercise 03, SWE-bench trajectories from exercise 02, Inspect eval logs from exercise 05, or a public agent-trace dataset (see below).

## Trajectory source

Any of these work:

- **Your exercise-03 traces.** Best option if you did WebArena or GAIA; the trajectories are agent-shaped and long.
- **Your exercise-02 SWE-agent traces.** Excellent for a rubric focused on code-repair; the trajectories are hundreds of lines and reward a careful rubric.
- **Public agent-trace corpus.** WebArena publishes reference-agent traces; OpenAgents publishes recorded sessions; several papers release `.jsonl` trajectories alongside their code. Pick a public source and cite it.
- **Synthetic trajectories from your Inspect eval (exercise 05).** Fine, as long as the sample is heterogeneous — you need trajectories that vary in quality so the rubric can discriminate.

Whichever you pick, ensure the 40–80 sample contains a *mix* of clear successes, clear failures, near-misses, and long-running failures. A rubric audit on 80 clean successes tells you nothing about the rubric's discriminative power.

## Requirements

### Part A — design the rubric

Ship `rubric.md` with a rubric that combines *at least two* of the three Chapter 5 shapes:

- **A milestone checklist** for tasks that decompose (2–5 boolean predicates per task template). Not required if your trajectories are open-ended research (GAIA-style).
- **A multi-dimensional anchored rubric** with 3–5 dimensions, each on a 0–4 scale, each with an anchor description per score point.
- **A free-form critique** with a fixed post-processing template that extracts a categorical `primary_failure_mode` label from a controlled vocabulary.

Include:

- A precise definition of each dimension. "Efficiency" is not a definition; "the trajectory reached the correct answer with a small number of tool calls relative to the task's minimum" is.
- The exact judge prompt (system + user template).
- The controlled vocabulary for failure modes (a fixed list — no free-form category names).

### Part B — build the human gold set

Sample **40 trajectories** stratified across quality — 10 clear-success, 10 clear-failure, 10 near-miss (partial credit expected), 10 long-running failures (budget-exhausted or thrashing). If your source is smaller than 40, use all of it.

Recruit **2 human raters** and have both independently score all 40 trajectories against your rubric. If you cannot recruit a second human, do the rating yourself twice with a 24-hour gap and note the caveat — self-agreement is a weaker signal than inter-annotator agreement, but on a 3-hour exercise it is often the only option.

Emit:

- `gold/labels.csv` — one row per trajectory per rater, columns for each rubric dimension.
- Inter-annotator agreement per dimension: Cohen's κ for the two-rater case, or Krippendorff's α if you generalize to 3+.
- For any dimension where κ < 0.5, revise the anchors and re-label. Re-labelling is part of the exercise, not a failure — a rubric that does not achieve human-human agreement cannot be audited against a judge.

Ship a resolved gold set (`gold/resolved.csv`) with an adjudicated score per (trajectory, dimension) where the raters disagreed. Follow mod-102 Chapter 4's adjudication guidance (majority / expert override / discussion).

### Part C — run the judge

Configure an LLM judge with the rubric prompt from Part A. Pick a judge model *distinct from the agent model that produced the trajectories* (mod-105: self-preference bias). Run the judge over the same 40 trajectories in the gold set, plus the remaining 0–40 unlabelled trajectories.

Log per-trajectory:

- Each dimension's score.
- The critique text.
- The extracted failure-mode label.
- Judge model, judge prompt hash, timestamp.

### Part D — the audit

Compute, per rubric dimension:

- **Judge-vs-adjudicated-human κ** (Cohen's κ, or weighted κ if the scale is ordinal 0–4 — use quadratic weights).
- **Confusion matrix** (adjudicated human anchor vs judge anchor). The matrix is more informative than the aggregate κ.
- **Per-anchor accuracy** (fraction of trajectories where the judge assigned the same anchor as the human).
- **Pairwise ranking agreement** — for every pair of trajectories in the gold set, does the judge rank them the same way the human does on this dimension? This is often more forgiving than exact-anchor agreement and is a better fit if the downstream use is ranking rather than absolute-score reporting.

Compute, for the failure-mode critique:

- **Failure-mode confusion matrix** — human failure-mode label vs judge failure-mode label.
- **Recall of the top-1 failure mode** — for each trajectory, does the judge's label match the human's?

### Part E — the decision, per dimension

For every rubric dimension and for the failure-mode critique, decide one of:

- **Use.** κ ≥ 0.6 (mod-105 threshold for decision-quality judge). The dimension can be reported in a headline number.
- **Re-anchor.** 0.4 ≤ κ < 0.6, or one anchor pair confuses systematically. The dimension is not usable as-is; revise anchors, re-audit. Note in the report which anchors are being revised and why (evidence from the confusion matrix).
- **Do not use.** κ < 0.4. The judge cannot score this dimension; either the dimension is under-specified for a judge to score, or the judge is not capable enough. Report this as a negative result; do not include the dimension in downstream reports.

State the decision per dimension in `docs/decisions.md` with the evidence.

### Part F — the run bundle

Ship:

- `rubric.md` — the rubric spec (Part A).
- `gold/labels.csv`, `gold/resolved.csv`, `gold/adjudication_notes.md` — the human gold set.
- `judge/scores.jsonl` — per-trajectory judge outputs.
- `judge/prompt.txt` — the exact judge prompt (system + user template, no interpolation).
- `audit/report.md` — the audit findings: per-dimension κ, confusion matrices, per-anchor accuracy, decisions.
- `MANIFEST.md` — judge model, judge prompt hash, gold set size, sampling method, rater identifiers (anonymized), adjudication method.
- `README.md` — one-shot rerun instructions.

## Starter guidance

- **Start with milestones on a subset if possible.** If any task templates in your source decompose cleanly, milestone predicates are the cheapest signal to author and the easiest to human-audit. Reserve the multi-dimensional anchored rubric for the dimensions that can't be predicate-checked.
- **Anchor writing is the whole game.** A rubric where anchors 2 and 3 differ only in adjectives ("mostly correct" vs "generally correct") will produce κ < 0.4 with any judge and often even with humans. Anchors should differ in behaviorally-observable ways ("agent recovered from at least one tool error" vs "agent never encountered a tool error").
- **Run a 5-trajectory pilot with humans before the full 40.** Iterate on the rubric anchors after the pilot. It is far cheaper to spend 30 minutes tightening the rubric than to run 40 labels through a rubric that will need to be re-run.
- **Pick a different judge model than the agent.** If your agent traces came from `gpt-4o`, use `claude-3-5-sonnet` (or a distinct model) as the judge. mod-105's self-preference finding is robust; do not pretend it doesn't apply here.
- **Do not use a 0–10 scale.** A 0–4 scale with anchors is the mod-105 recommendation. 0–10 without anchors will produce judge scores that regress to 6 and destroy your κ.
- **Report a negative result if the judge fails.** A "do not use" decision on a dimension is a legitimate outcome. Rubrics that turn out to require a human are the ones that most need to be flagged before they get published as if they were judge-scorable.
- **Log the judge model version.** Judge models change; a κ measured against `claude-3-5-sonnet-20240620` is not automatically valid against `claude-3-5-sonnet-20241022`. The manifest is the pin.

## Acceptance criteria

- The rubric combines at least two of the three Chapter 5 shapes and has anchored descriptions for every score point on every dimension.
- The human gold set has at least 40 trajectories, stratified by quality, dual-labelled, with adjudicated resolutions.
- Inter-annotator agreement per dimension is computed and reported; dimensions with κ < 0.5 were re-anchored before the judge audit.
- Judge outputs are logged with the judge model, prompt hash, and full critique text.
- The audit report computes judge-vs-human κ (weighted for ordinal scales), a confusion matrix, and per-anchor accuracy per dimension.
- A `use / re-anchor / do-not-use` decision is stated per dimension with evidence.
- The manifest pins the judge model, prompt, gold set version, and adjudication method.

## Stretch goals

- **Order and length bias probes.** Run the judge with trajectories presented in reversed order for the pairwise ranking check; report position-bias magnitude. Truncate 10 trajectories to their first N messages (preserving success/failure ground truth) and re-run the judge; report length-bias magnitude.
- **Two-judge ensemble.** Run a second judge model. Compute inter-judge κ; treat trajectories where the two judges disagree as "high-uncertainty" and route them to human review in a hypothetical production pipeline.
- **Rubric-drift audit template.** Write a small script that takes the same gold set and a new judge model version and re-runs the audit, producing the same report. This is the artefact your future self needs when you upgrade the judge model.
- **Per-slice κ.** Compute agreement per task category or per source model. A judge that is well-calibrated on WebArena tasks but poorly calibrated on GAIA tasks needs to be scoped by slice, not deployed broadly.
- **Turn the rubric into an Inspect scorer.** Wire the audited rubric into the exercise-05 Inspect eval as a scorer that reads the trajectory, calls the judge, and emits a `Score` with per-dimension metadata. Verify against a hand-picked trajectory that the resulting score matches the standalone rubric run.
