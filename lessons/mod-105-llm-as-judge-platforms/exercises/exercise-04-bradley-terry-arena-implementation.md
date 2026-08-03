# exercise-04: Bradley-Terry Arena Implementation

**Estimated effort:** 3 hours

## Objective

Stand up a working Arena-style pairwise rating system for a small population of candidate models: a Bradley-Terry maximum-likelihood fit with cluster-bootstrap confidence intervals, an ELO update as a live-rating comparison, and a rating-proximal match sampler. Demonstrate the statistical properties you should be able to reason about from Chapter 6 — CIs shrinking with match volume, rating-proximal sampling reducing pairs-to-CI-target, sensitivity to tie convention, and the ranking's dependence on judge quality. The deliverable is a reproducible arena harness plus a report that connects each property to a plot.

## Prerequisites

- mod-105 Chapters 3, 4, 6 (pairwise rubrics, bias controls, Bradley-Terry / ELO).
- exercise-01 (a pairwise rubric you trust) and exercise-02 (bias-controlled harness) are strongly recommended.
- Access to at least one judge model that can grade your pairwise rubric.
- A candidate pool of at least 5 subject systems that respond to a shared prompt set. If you don't have that, generate it: pick 5 models spanning tiers (small OSS, mid OSS, large OSS, hosted mid-tier, hosted frontier) and run a shared 100–200 item prompt set through each. Anonymize model identity in the arena — judges must not see system names.

## The setup

- **Systems:** 5 or more candidate systems, anonymized (`sys_01`, `sys_02`, ...).
- **Prompt set:** 100–500 items, ideally from a real distribution.
- **Judge:** one judge with bias controls (swap-and-average) from exercise-02. For a stretch goal, add a second judge.
- **Match log:** every judged pair is one row: `pair_id`, `prompt_id`, `system_a`, `system_b`, `verdict ∈ {A, B, tie}`, `consistent_after_swap`, `judge`, `timestamp_index` (an ordinal, since `Date.now()` is unavailable in the workflow harness — increment per match).

Ties are common and load-bearing. Use the half-wins convention (`tie` = 0.5 wins each side) unless you have a specific reason to do otherwise, and document the choice in the report.

## Requirements

### Part A — the match generator

Ship `arena/match.py` implementing two match schedulers:

1. **Uniform-random pairing.** Given N systems, sample pairs uniformly at random over the `N*(N-1)/2` distinct unordered pairs. This is the baseline.
2. **Rating-proximal pairing.** After an initial N_bootstrap matches sampled uniformly, sample subsequent pairs weighted by proximity of current rating estimates (e.g., inverse of `|R_i - R_j|` with a temperature). Use whatever proximity heuristic you like; document it.

For both schedulers, apply swap-and-average from exercise-02 for every pair. Log both order-1 and order-2 verdicts and the collapsed verdict.

### Part B — the Bradley-Terry fit

Ship `arena/bt.py` implementing `fit_bradley_terry(matches, systems) -> {system: rating, system: (lo, hi)}` following Chapter 6:

- Fit via logistic regression with one-hot per-system features. `sklearn.linear_model.LogisticRegression(penalty="l2", C=1e6, fit_intercept=False)`.
- Pin one system's rating to 0 (drop its column) so ratings are identified.
- Rescale ratings to an ELO-like scale where a 100-point gap ≈ 64% win rate (multiply logits by `400 / ln(10)`).
- Handle ties by the two-row expansion (a tied pair `(a, b)` contributes two rows with y = 1 and y = 0). Document the choice.

**Cluster-bootstrap CIs.** Resample by prompt (not by match) with replacement, refit, and take the empirical 2.5 / 97.5 percentiles of each system's rating. 500–1000 resamples. If you resample by match instead of by prompt, note in the report that the CIs are optimistic and explain why.

### Part C — the ELO comparator

Ship `arena/elo.py` implementing an online ELO update following Chapter 6:

- Initial rating 1000 for every system.
- K = 32 for the first N_bootstrap matches per system, K = 16 thereafter (chess convention analog for early volatility).
- Update after each match in log order.
- Save per-match rating snapshots so you can plot rating trajectory over time.

### Part D — the experiments

Run four experiments and produce a plot for each.

**Experiment 1: CI shrinkage with match volume.** Under uniform-random pairing, fit Bradley-Terry after 100, 200, 400, and 800 matches. Plot each system's rating with 95% CI as a function of match count. The CIs should shrink roughly with `1 / sqrt(N)`.

**Experiment 2: Uniform vs. rating-proximal sampling.** Run 400 matches under each scheduler. Compare the CI on the *tightest* rating-difference pair (the two systems whose ratings are closest). The rating-proximal scheduler should produce a tighter CI on that difference at the same total match count. Quantify the improvement.

**Experiment 3: BT vs. ELO.** Overlay Bradley-Terry ratings (batch, refit after all matches) with ELO ratings (final, after the same match sequence). Report the ranking agreement (Spearman ρ between the two rating vectors) and the pointwise rating deltas. They should be close but not identical.

**Experiment 4: Tie-convention sensitivity.** Refit Bradley-Terry under three tie conventions: half-wins, drop tied matches, and a Davidson tie model (stretch: only if you want to). Report the per-system rating and the per-pair CI under each. If the ranking changes across conventions, name the pair whose ordering is unstable and how much it moved.

Each experiment gets its own subsection in the report with the plot and a paragraph of interpretation.

### Part E — the report

Write `REPORT.md` (≤ 3 pages) covering:

1. **Setup.** Systems, prompt set, judge, rubric, bias controls in place.
2. **Match log summary.** Total matches, ties, swap-inconsistent pairs, per-system match count.
3. **Final leaderboard.** Bradley-Terry ratings with 95% CIs. Explicitly call out any pairs of systems whose CIs overlap ("indistinguishable at current match volume").
4. **Experiment 1** — CI shrinkage plot + one paragraph.
5. **Experiment 2** — uniform vs. rating-proximal plot + one paragraph.
6. **Experiment 3** — BT vs. ELO comparison + one paragraph.
7. **Experiment 4** — tie-convention sensitivity + one paragraph.
8. **Judge-quality ceiling.** Cite the swap-inconsistency rate from your log as the judge-quality diagnostic (Chapter 6: rating quality is ceilinged by judge accuracy on each pair). Reason briefly about how much of the residual CI is judge noise vs. sample size.
9. **What would you change with 5× the budget?** More matches, more judges, a panel, more systems? Ground the answer in what tightened in the experiments and what didn't.

### Part F — bundle

Ship:

- `data/prompts.jsonl`, `data/responses/*.jsonl` (per-system responses)
- `arena/match.py`, `arena/bt.py`, `arena/elo.py`, `arena/plot.py`
- `logs/matches.jsonl` — the full match log
- `results/plots/*.png` — one per experiment
- `results/leaderboard.md` — the final BT leaderboard with CIs
- `REPORT.md`
- `run.sh` — one-shot rerun of the arena (assumes responses already generated)

## Starter guidance

- **Anonymize systems in the judge prompt.** Every pairwise judgment must see `Response A` / `Response B`, not `Response from gpt-4o`. This is the same discipline as exercise-02. Without it, self-preference and priors contaminate the rating.
- **Swap-and-average is not optional for arenas.** Uncontrolled position bias systematically inflates whichever position the judge prefers, which shifts every rating. Chapter 4 + exercise-02 belong upstream of this exercise.
- **Cluster-bootstrap by prompt, not by match.** Matches from the same prompt share the prompt's difficulty and any judge quirks on that prompt. Resampling by prompt respects that clustering; resampling by match assumes independence and gives you optimistic CIs. This alone can be a 2× effect on CI width.
- **Do not stop when the ratings "look right."** The natural failure mode of arena-building is convergence bias: you stop adding matches when the ranking matches your intuition, which cherry-picks a lucky sample. Preregister the total match budget or the CI target, and stop when *that* is met, not when the ranking pleases you.
- **Rating-proximal sampling needs an initial random phase.** With a cold start, every rating is at the pinned value and "proximal" is meaningless. Run 100–200 uniform matches before switching to rating-proximal. Document the switch.
- **ELO's early ratings are volatile.** The K = 32 phase for the first 20 matches per system is essential; a system whose first match is against the strongest player will drop 50 points that never come back with small K.
- **Half-wins for ties is the default for a reason.** It composes cleanly with logistic-regression Bradley-Terry, has a well-defined ELO analog, and is what Chatbot Arena publishes. Move away from it only for a specific reason — Davidson tie models are principled but overkill for most arenas.
- **Rating-difference is the meaningful quantity.** A ratings table with everyone shifted by +200 is the same leaderboard. Report differences (or per-pair predicted win rates) alongside absolute ratings; readers who don't know the convention will over-interpret a "1420" as absolute skill.
- **When your CI plots look weird, check the match log for a system that played mostly one other system.** Bradley-Terry needs *coverage*. A system with 100 matches all against `sys_03` has huge uncertainty against every other system; the CI plot will reveal it.
- **Do not conflate "the arena's ranking is wrong on this pair" with "the aggregator is broken."** The aggregator faithfully aggregates the judge's verdicts. If the ranking is wrong, the judge is wrong on those pairs — go debug the judge with exercise-03's calibration slice, not the fitter.

## Acceptance criteria

- Bradley-Terry fit is implemented from a match log and produces per-system ratings with cluster-bootstrap 95% CIs. Ties are handled with a documented convention (half-wins by default).
- ELO update is implemented and processed in match order; per-match rating snapshots are saved.
- Match scheduler supports both uniform-random and rating-proximal modes with a documented switch condition.
- Swap-and-average is applied to every pair (or it is explicitly and prominently noted why not, for a specific experiment).
- Four experiments (CI shrinkage, uniform vs. proximal, BT vs. ELO, tie-convention sensitivity) are run and each has a plot + a paragraph of interpretation.
- The leaderboard explicitly flags any system pairs whose CIs overlap as statistically indistinguishable at the current match volume.
- The report ties residual CI width to *both* sample size and judge accuracy, not just sample size.
- The bundle is reproducible: a reviewer can rerun the arena via `run.sh` and reproduce the leaderboard within stochastic tolerance.

## Stretch goals

- **Two-judge panel.** Add a second judge (from a different family) and run every match through both. Aggregate by majority vote per pair. Refit Bradley-Terry on the panel verdicts and compare against each single-judge fit. Report the ranking shifts and CI changes. This is the empirical case for panels.
- **Judge-noise ablation.** Add controlled judge noise: flip a random ε fraction of verdicts before fitting. Plot rating CI width as a function of ε to demonstrate the "judge accuracy is the ceiling" property numerically.
- **Match-scheduler ablation.** Add a third scheduler — Swiss-tournament style, where a system that just won is matched against a higher-rated opponent — and show that the ratings are biased under this scheduler when you fit vanilla Bradley-Terry. Point at the Chapter 6 note on outcome-conditioned sampling.
- **Live-updating ELO with a decay.** Add an exponential decay on rating (e.g., regress-to-mean at 0.5% per week) and demonstrate that a system that stopped playing months ago gradually returns to the mean. This mirrors what real arenas do to handle model deprecation.
- **Rating-comparison markdown.** Auto-generate a `results/head_to_head.md` matrix showing, for every pair of systems, the predicted win rate from the fitted BT model alongside the *observed* win rate in the match log. Large discrepancies are model-fit diagnostics — pairs where the model doesn't fit the observed rate suggest one of the systems has a non-transitive relationship to the others.
- **Krippendorff's α across the panel.** If you did the two-judge panel stretch, compute Krippendorff's α across all judges + humans on a small held-out slice. This is a stronger check on the panel's quality than pairwise agreement.
