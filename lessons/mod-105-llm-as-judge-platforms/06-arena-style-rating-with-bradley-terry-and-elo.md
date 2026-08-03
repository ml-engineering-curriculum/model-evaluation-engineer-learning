# Arena-Style Rating With Bradley-Terry And ELO

Pairwise judging gives you per-pair verdicts. What it does not give you, on its own, is a *system rating* — a single number per system that ranks it against every other system. Sparse coverage makes the raw win rate misleading (a system that only played easy opponents will look artificially strong), and O(N²) pair coverage is expensive to fill in exhaustively as the number of systems grows. Arena-style leaderboards solve this by fitting a *rating model* on top of the pairwise verdicts. This chapter is about the two dominant rating models — Bradley-Terry and ELO — how they work, how they are fit, and what they promise and do not promise about the resulting ranking.

## The problem the rating model solves

Suppose you have five systems (A, B, C, D, E) and 500 pairwise verdicts distributed unevenly across the 10 possible pairs. Raw per-system win rate has three problems:

1. **Opponent-strength bias.** A system that mostly played the weakest opponents has a high win rate for a reason that has nothing to do with its ability.
2. **Coverage sparsity.** Two systems that never played each other have no direct pairwise information.
3. **No uncertainty.** A system with 10 pairs behind it and a system with 500 pairs behind it get the same "one point" on the leaderboard, with no indication that one is far better measured.

A rating model turns the sparse pairwise verdicts into a per-system latent skill parameter with an uncertainty. The two standard ones for arena-style leaderboards are Bradley-Terry (a static maximum-likelihood fit over the full match history) and ELO (an online rating update, one match at a time). They give similar-but-not-identical rankings, and each is more natural in a specific setting.

## Bradley-Terry: the static maximum-likelihood fit

The Bradley-Terry model (Bradley and Terry 1952) is the standard parametric model for pairwise comparison data. It assigns each system `i` a latent skill parameter `θᵢ`, and models the probability that system `i` beats system `j` as:

```
P(i beats j) = exp(θᵢ) / (exp(θᵢ) + exp(θⱼ))
             = 1 / (1 + exp(θⱼ - θᵢ))
```

The rating for each system is fit by maximum likelihood over the full set of match outcomes. This is a *logistic regression* problem: for each pair `(i, j)` with `wᵢⱼ` wins for `i` and `wⱼᵢ` wins for `j`, the log-likelihood contribution is:

```
ℓᵢⱼ = wᵢⱼ * log σ(θᵢ - θⱼ) + wⱼᵢ * log σ(θⱼ - θᵢ)
```

where `σ` is the sigmoid. The θs are only identified up to an additive constant (adding 10 to every θ leaves all match probabilities unchanged), so one system is pinned to θ = 0 as a reference, or a small L2 penalty is added to keep the fit stable.

In code (scikit-learn's `LogisticRegression` handles the whole fit with one-hot features):

```python
import numpy as np
from sklearn.linear_model import LogisticRegression

def fit_bradley_terry(matches, systems):
    """
    matches: list of (winner, loser) tuples over pairwise verdicts.
             Ties are handled by expanding a tie between (a, b) into
             two rows: (a, b) with outcome 0.5 and (b, a) with outcome 0.5.
    systems: list of system names.
    Returns: dict {system: rating}.
    """
    n = len(systems)
    idx = {s: i for i, s in enumerate(systems)}

    X_rows, y_rows = [], []
    for winner, loser in matches:
        x = np.zeros(n)
        x[idx[winner]] = 1
        x[idx[loser]] = -1
        X_rows.append(x)
        y_rows.append(1)

    X = np.array(X_rows)
    y = np.array(y_rows)

    # Pin system 0's rating to 0 by dropping its column and offsetting.
    model = LogisticRegression(
        fit_intercept=False,
        penalty="l2",
        C=1e6,           # weak regularization; large C = weak penalty
    ).fit(X, y)

    return {s: model.coef_[0, idx[s]] for s in systems}
```

The chatbot-arena `arena-hard-auto` and `fastchat` repos ship reference implementations of this fit with production niceties (bootstrap confidence intervals, tie handling with the two-row expansion above, sparse solver for many-system arenas).

**What Bradley-Terry gives you.** Per-system ratings on an arbitrary scale (usually rescaled to an ELO-style scale where a 100-point difference corresponds to a specific win probability), with a bootstrap confidence interval per rating produced by resampling matches with replacement and refitting.

**What it does not give you.** A meaningful *magnitude* of the rating — only the differences are meaningful. And it assumes match outcomes are independent, which is not strictly true for judge-based pairs (the same judge scoring many pairs induces correlation), so bootstrap CIs are optimistically narrow in practice. Cluster-bootstrap by judge or by prompt to get honest CIs.

## ELO: the online updating rating

ELO (Elo 1978, originally for chess) is the online cousin of Bradley-Terry. It maintains a per-system rating `Rᵢ` and updates it after every match. The predicted probability that `i` beats `j` is:

```
P(i beats j) = 1 / (1 + 10^((Rⱼ - Rᵢ) / 400))
```

The update after a match with observed outcome `Sᵢ ∈ {0, 0.5, 1}` and predicted probability `Pᵢ`:

```
Rᵢ ← Rᵢ + K * (Sᵢ - Pᵢ)
Rⱼ ← Rⱼ + K * ((1 - Sᵢ) - (1 - Pᵢ))
```

where `K` is a step size (chess uses K = 32 for casual, K = 16 for master-level).

```python
def elo_update(ratings, winner, loser, K=32, tie=False):
    """
    In-place ELO update. `ratings` is a dict {system: rating}, initial 1000.
    Winner/loser are system names; for a tie, pass tie=True and either
    order (result is symmetric).
    """
    Rw, Rl = ratings[winner], ratings[loser]
    Pw = 1 / (1 + 10 ** ((Rl - Rw) / 400))
    Pl = 1 - Pw
    Sw, Sl = (0.5, 0.5) if tie else (1.0, 0.0)
    ratings[winner] = Rw + K * (Sw - Pw)
    ratings[loser] = Rl + K * (Sl - Pl)
```

**ELO's advantage** is that it is online and cheap — one addition per match, no full refit — which is what made it viable for chess and what makes it viable for a live leaderboard that gets thousands of new matches per day.

**ELO's disadvantage** relative to Bradley-Terry is order-dependence: the rating you get depends on the order the matches were processed in, and early matches move a rating more than late ones (until enough matches have accumulated to stabilize it). The bootstrap-CI story is also messier — you can't just resample matches and rerun, because order matters. Chatbot Arena originally used ELO and has since moved to a Bradley-Terry fit as the *published* rating precisely because Bradley-Terry is more robust to match ordering, though ELO is still useful for a live-ish estimate between refits.

## The relationship between the two

Bradley-Terry and ELO are the same underlying model expressed differently. The Bradley-Terry log-odds is the ELO rating difference divided by 400 (times ln(10)); the two ratings can be converted by:

```
elo_rating(system_i) = 400 * θᵢ / ln(10) + offset
```

If you fit Bradley-Terry to steady-state match data, then run ELO with a small K on the *same* matches, they will converge to nearly the same ranking. The differences are practical: Bradley-Terry is a batch estimator with clean uncertainty; ELO is an online estimator with volatile early ratings. Use Bradley-Terry for publishable leaderboards. Use ELO for live displays.

## Match scheduling: not all pairs are equally informative

The naive strategy — every system plays every other system equally often — is O(N²) and wasteful when some systems are clearly stronger than others. The information-gain optimum is to play pairs whose current rating estimates suggest a close outcome; pairs where one system dominates give you a predictable win and little information.

Two practical strategies:

- **Uniform-random pairing.** Simple, unbiased, wasteful. Fine for small arenas (N < 10).
- **Rating-proximal pairing.** Bias the sampler toward pairs whose current estimated ratings are close. Reduces total matches needed to achieve a target CI on rating differences by 30–60% in practice. Requires an initial random-pairing phase to bootstrap ratings.

Chatbot Arena uses a rating-proximal sampler; `arena-hard-auto` documents the strategy in its repo.

**Do not condition the sampler on match outcomes in a way that leaks.** A sampler that "matches challengers against the current top model until they lose" is a *Swiss-tournament* schedule and gives biased ratings without a matching correction in the fit. Standard rating-proximal sampling based on rating *estimates before the match* is fine; adaptive sampling based on the outcome of the match you are about to fit on is not.

## Ties: pick a convention and be consistent

The ternary verdict space (A / B / tie) creates a modeling decision. Three standard treatments:

- **Half-wins (the ELO convention).** A tie counts as 0.5 wins for each side. Simple; consistent with the way ELO updates ties.
- **Drop tied matches.** Some Bradley-Terry variants drop tied matches from the fit entirely, on the argument that they carry no information about which system is stronger.
- **Explicit tie model (Davidson 1970).** Extend Bradley-Terry with a third parameter modeling the probability of a tie as a function of the rating gap. Most principled, more complex to fit, standard in the sports-analytics literature; not commonly used in LLM arenas.

The half-wins convention is what Chatbot Arena and `arena-hard-auto` use. Pick a convention, document it, and be consistent — the ranking can shift under a different convention when many pairs are ties.

## Statistical properties you should be able to reason about

**Rating differences, not rating levels, are meaningful.** A 100-point ELO gap corresponds to roughly 64% expected win rate for the higher-rated side (from the ELO formula: `1 / (1 + 10^(-100/400)) ≈ 0.640`). Levels are on an arbitrary offset; the community convention of centering the strongest historical model around 1400–1600 is just a display choice.

**Confidence intervals require the bootstrap.** Parametric CIs from the Bradley-Terry likelihood are optimistic because match outcomes are not independent (same judge on many pairs, same prompt across many pairs). Use a cluster bootstrap — resample by judge, by prompt, or by day — and refit. The `arena-hard-auto` repo does a per-prompt bootstrap by default.

**Two systems have "statistically distinguishable" ratings when their CIs do not overlap.** A system at 1250 (CI 1220–1280) and a system at 1240 (CI 1210–1270) are not distinguishable, regardless of the point estimates. Leaderboards that show ranks without CIs are trading precision for false confidence.

**Ratings drift when the population of opponents drifts.** A system's rating goes up if the *other* systems get weaker, even if the system itself did not change. This is why arena leaderboards periodically re-anchor to a fixed reference model, and why comparing this month's #3 to last quarter's #3 is meaningful only if both were rated against the same reference set.

**Judge quality upper-bounds arena quality.** If the underlying judge (human or LLM) has 75% accuracy on the pairwise task, no rating model can do better than the ranking implied by 75%-accurate outcomes. Chapter 5's judge-calibration exercise is the ceiling; the rating model just aggregates efficiently under that ceiling. This is why "the arena's ranking is wrong on this pair" is often actually "the judge is systematically wrong on this pair type," and the fix is at the judge or rubric level, not the aggregator.

## A minimum-viable arena

For an internal head-to-head bake-off across 5–15 candidate systems, the minimum-viable arena is:

1. A fixed prompt set of 100–500 items covering your task's distribution.
2. A pairwise rubric (Chapter 3) with position-bias control (Chapter 4) applied on every pair.
3. Rating-proximal sampling after an initial round of uniform-random pairing to bootstrap ratings.
4. A Bradley-Terry fit over the full match log after each round of comparisons, with per-prompt bootstrap CIs.
5. A leaderboard display that shows *rating with CI*, *number of matches*, and *rating change since last refit* per system.
6. A per-pair swap-inconsistency rate published alongside as a judge-quality diagnostic.

That is a working arena. The published Chatbot Arena is a scaled-up version with human votes as the primary signal, GPT-4 as a proxy judge for extra coverage, and continuous rating refits — but the mechanics are the same.

## Summary

Bradley-Terry is the standard maximum-likelihood rating model for pairwise comparison data: assign each system a latent skill, fit by logistic regression, get per-system ratings with bootstrap-CI uncertainty. ELO is the online cousin — same underlying model, one-match-at-a-time updates — cheap and live but order-dependent. Match scheduling should favor rating-proximal pairs after an initial random-pairing phase. Ties get a documented convention (half-wins is standard). Two ratings are meaningfully different only when their CIs do not overlap, and CIs require a cluster bootstrap because pairwise outcomes are not independent. The arena's ranking is ceilinged by the judge's per-pair accuracy — the aggregator does not fix a broken judge. The next chapter closes the module by choosing the judge itself: hosted frontier, open-source, or dedicated-trained, and how to route items across tiers under a cost budget.
