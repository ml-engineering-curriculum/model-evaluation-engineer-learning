# exercise-02: Position And Length Bias Controls

**Estimated effort:** 3 hours

## Objective

Measure the three systematic judge biases from Chapter 4 — position, length, and self-preference — on a real judge, then implement the standard controls and quantify what they buy you. The deliverable is a *bias-controlled pairwise judging harness* plus a report that shows, for one specific judge on one specific task, how much each control reduces bias and how much residual bias remains after controls.

You are not designing rubrics in this exercise (use one from exercise-01 or a published rubric such as the MT-Bench pairwise prompt). You are measuring and mitigating.

## Prerequisites

- mod-105 Chapters 3 and 4 (pairwise rubrics, bias controls).
- exercise-01 completed (you have a pairwise rubric you trust) *or* a published rubric you can point to.
- Access to at least two judge models from different families (for the self-preference diagnostic). Reasonable pairs: a hosted frontier judge + an OSS instruction-tuned judge; two hosted judges from different providers.
- A candidate set of at least 3 subject models that generated responses to a shared prompt set. If you don't have that on hand, generate it as part of the exercise (below).

## The dataset

You need a *pair dataset* of the shape `(prompt, response_from_system_x, response_from_system_y)` covering multiple system pairs. Two options:

**Option A: use a published pair dataset.** Chatbot Arena releases anonymized pairwise conversation data periodically; MT-Bench and Arena-Hard-Auto both ship prompt sets. Pick a subset of 100–200 pairs covering at least 3 distinct source systems.

**Option B: generate your own.** Take a prompt set of 100–200 items (from exercise-01, from MT-Bench, or from your own task). Generate a response from each of 3 candidate subject models (e.g., a large model, a small model of the same family, and a model from a different family). Then materialize all `(prompt, sysX_reply, sysY_reply)` pairs — with 3 systems this is 3 pairs per prompt (AB, AC, BC).

Ship the dataset as `data/pairs.jsonl` with fields: `pair_id`, `prompt`, `system_a`, `response_a`, `system_b`, `response_b`, `len_a`, `len_b`. Anonymize model identity — the judge must not see system names.

## Requirements

### Part A — baseline: no bias controls

Run the judge on every pair *without* controls. Record: `pair_id`, `judge_model`, `verdict`, `raw_output`.

Compute:

- Overall win rates per system.
- Per-pair verdict.

This is the "how the judge sees the world with no controls" baseline. Ship as `logs/baseline.jsonl` and `results/baseline.md`.

### Part B — position-bias control (swap-and-average)

Implement `swap_and_average(judge, pair) -> (verdict, consistent_flag)` following Chapter 4:

- Judge each pair in both orders.
- If verdicts agree (after relabelling), keep the verdict.
- If they disagree, collapse to `tie` and set `consistent_flag=False`.

Rerun the judge over all pairs with the swap-and-average controller. Ship as `logs/swapped.jsonl` and compute:

- **Swap-inconsistency rate** — fraction of pairs where the judge flipped when position was swapped. This is a first-class metric; publish it prominently.
- **Ranking delta** — how did per-system win rates change vs. baseline? Any system whose ranking moved is evidence that baseline was position-biased.
- **Cost multiplier** — swap-and-average doubles judge calls. Report the actual cost delta.

### Part C — length-bias diagnostic and control

Two sub-parts.

**Sub-part 1: measurement.** From the swapped log, compute the length-conditional win rate — the fraction of consistent (non-tie) wins that go to the *longer* response. On a well-calibrated judge over your pair distribution, this should be close to whatever the true rate is. If your dataset has no ground truth on length-independence, construct a **length-bias probe subset**: 20–30 pairs where the two responses are known to be equivalent quality but differ meaningfully in length (rewrite short responses to be verbose without changing content, or truncate long responses to their core content). On this probe subset, the correct long-wins rate is ~50%; any deviation is the length bias.

Report:

- Longer-wins rate on the full dataset (context, not proof of bias).
- Longer-wins rate on the probe subset (this is the bias measurement — a value like 68% is a real bias).
- Correlation between length delta `|len_a - len_b|` and probability the longer side wins.

**Sub-part 2: control.** Implement two mitigations and measure each:

1. **Rubric-side control:** add the anti-length line ("Do not prefer a longer response merely because it is longer; prefer the one that better addresses the criteria above") to the rubric and rerun. Report the new longer-wins rate on the probe subset.
2. **Aggregation-side control:** compute a length-controlled win rate à la AlpacaEval 2 — stratify pairs into length-similar buckets and report the win rate per bucket, then report a length-adjusted overall win rate as the weighted mean across buckets. Publish both raw and length-controlled numbers.

### Part D — self-preference diagnostic

Run the same swapped, length-controlled eval with a **second judge model from a different family**. Compare:

- Per-system win rate under judge 1 vs. judge 2.
- Judge-vs-judge agreement (fraction of pairs where the two judges' verdicts match).
- **Self-preference indicator:** for each judge, compute the win rate of *responses from that judge's own family* (if any of your subject systems share a family with the judge). If judge 1's family systems win more under judge 1 than under judge 2, that gap is a self-preference measurement.

If you can afford a third judge, run it — three-judge panels give you majority-vote resolution on the disagreements and a much cleaner self-preference number.

### Part E — the report

Write `REPORT.md` (≤ 3 pages) covering:

1. **Judge and dataset.** Which judge(s), which prompt set, which subject systems, how the pairs were generated.
2. **Baseline results.** Per-system win rates without controls.
3. **Position-bias measurement.** Swap-inconsistency rate; per-system win-rate deltas after swap-and-average.
4. **Length-bias measurement.** Longer-wins rate on the probe subset before and after each control; correlation between length delta and winner.
5. **Self-preference measurement.** Judge-1 vs judge-2 per-system win-rate deltas; explicit call-out of any family-alignment effect.
6. **Recommendation.** For this judge on this task, which controls are load-bearing and which are cosmetic? What is the shipping configuration you would recommend to a teammate running this rubric next week?
7. **Residual bias.** After all controls, what bias do you *still* see? What would it take to reduce further?

### Part F — bundle

Ship:

- `data/pairs.jsonl`, `data/length_probe.jsonl`
- `judge/harness.py` — the bias-controlled harness (baseline mode, swap mode, length-probe mode, self-preference mode)
- `logs/baseline.jsonl`, `logs/swapped.jsonl`, `logs/length_probe.jsonl`, `logs/self_pref_judge2.jsonl`
- `results/*.md` — per-experiment tables
- `REPORT.md`
- `run.sh` — one-shot rerun of all experiments

## Starter guidance

- **Anonymize model identity in the pair records the judge sees.** The judge prompt should refer only to Response A / Response B; system provenance lives in your log metadata, not in the judge prompt. Failing this contaminates the self-preference measurement.
- **Do not use the same judge model as any of your subject systems.** Even if the judge is a "different version" of the same base model, self-preference bleeds. Pick a judge from a demonstrably different family.
- **Swap-and-average is not free — budget for it.** Every pair now costs two judge calls. On a 200-pair × 3-system-pair × 2-judge study, that is 2,400 judge calls. Do not blow the budget on a large N; a well-run 150-pair experiment with three controls is more informative than a 500-pair experiment with only baseline.
- **The length probe is the study's key subset.** If you cannot construct a probe subset where you *know* the correct answer is length-independent, you cannot cleanly measure length bias — you're stuck reporting correlations that could reflect either bias or a real quality signal. Spend the time to build a defensible probe subset.
- **Report swap-inconsistency rate prominently, not as a footnote.** It is a joint measure of judge quality and rubric ambiguity. A rate over 30% on close pairs means the judge cannot stably distinguish them, and any per-system ranking you draw from those pairs is noise.
- **Correlate length delta with winner even outside the probe subset.** Pearson correlation between `len_a - len_b` and `verdict == A` on the full dataset is a cheap sanity check. Correlations above ~0.3 in absolute value are a strong prior on length bias.
- **The self-preference test needs disjoint families.** Judge 1 = GPT-4, judge 2 = GPT-4-mini is not a self-preference test — they're the same family. Judge 1 = GPT-4-class, judge 2 = Claude-Opus-class, judge 3 = Llama-3-70B is a clean setup.
- **Ties matter more than they look.** Swap-and-average collapses inconsistent pairs to tie. If your rubric had a low tie rate at baseline and a high one after swap-and-average, most of your baseline "wins" were order-dependent. That is a real result, not an artifact.

## Acceptance criteria

- The judging harness supports at minimum: baseline mode, swap-and-average mode, and length-probe mode, invocable from `run.sh`.
- Swap-and-average is implemented correctly (verdict relabelling on the swapped order is verified against a hand-worked test case in the code).
- Swap-inconsistency rate, length-conditional win rate on the probe subset, and per-judge per-system win rate are all reported with numbers, not just qualitative claims.
- A length probe subset of 20–30 pairs where responses are known to be length-independent-in-quality is included and used in the length-bias measurement.
- A second judge from a different model family is run and the judge-vs-judge disagreement is reported.
- `REPORT.md` names, for this specific judge on this specific task, which controls actually moved the numbers and by how much.
- The bundle is reproducible: a reviewer with credentials for both judges can rerun the full experiment via `run.sh`.

## Stretch goals

- **Panel-of-three.** Add a third judge from a third family and run the same experiment. Aggregate by majority vote and compare per-system win rates against each single-judge result. Report the delta between the panel and each single judge — this is the empirical value of paneling.
- **Position-bias on generation order.** Some judges show a *last*-position bias rather than a first-position bias. Compute per-position win rate independently for each judge (rate at which the judge picks the response in position 1 across all pairs). Publish the number; it is a per-judge characteristic worth knowing.
- **Style-bias diagnostic.** Construct a second probe subset where two responses differ in style (formal vs. casual, bulleted vs. prose) but not content. Report the style-shifted win rate. This is not one of the "three canonical biases" but it is common and worth adding to any bias battery you build.
- **Bootstrap CIs on every reported rate.** Every win rate, agreement rate, and bias number in the report should carry a bootstrap 95% CI. A judge with a 62% ± 8% longer-wins rate is not statistically different from 50%; a judge with 62% ± 3% is.
- **Cost accounting.** Report the token cost of each control (swap-and-average doubles calls; panel-of-three triples them; anti-length rubric adds a few tokens per prompt). Present a table of `cost multiplier vs bias reduction achieved` so the shipping recommendation in Part E is grounded in the trade-off, not aesthetic preference.
