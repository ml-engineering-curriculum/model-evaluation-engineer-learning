# Controlled A/B Experiments with CUPED and Pre-Registered Hypotheses

Once the offline regression suite (Chapter 2) has passed and the shadow (Chapter 3) has run clean, the candidate model earns exposure to real users under a controlled experiment. This is the altitude at which the metrics that actually matter to the business — user thumbs-up rate, session continuation, downstream conversion, revenue-per-user, retention — become measurable, because for the first time a real user is seeing the candidate's response and reacting to it. Everything above this altitude was proxy; A/B is where the proxies get grounded.

A/B experimentation is a large field with a large literature, and this module does not attempt to cover it end-to-end. Kohavi, Tang, and Xu 2020 ("Trustworthy Online Controlled Experiments") is the reference book; every organization running experiments at scale has a version of the checklist that book codifies. This chapter is narrower: it covers the choices that are *specific to model releases*, the choices that are frequently gotten wrong on ML products, and one variance-reduction technique — CUPED (Deng, Xu, Kohavi, Walker 2013) — that is under-used in ML A/B setups and pays for itself on almost every design.

The three ideas the chapter builds:

1. **Pre-registration.** Before the experiment runs, write down the hypothesis, the metrics, the segmentation, the sample size, the decision rule, and the stopping criterion. This is not academic ritual; it is what defends the launch against post-hoc reasoning that lets favorable-looking noise ship.
2. **Randomization unit.** Requests, sessions, and users are all valid units, but each carries a different assumption about interference. Getting the unit wrong is the most common way an ML A/B gives a confidently wrong answer.
3. **Variance reduction via CUPED.** Given a covariate correlated with the outcome and unaffected by the treatment, the variance of the treatment-effect estimator can be reduced by 20–70% for free — cutting required sample size (and therefore experiment duration and exposure to a potentially worse candidate) by the same fraction.

## What A/B measures that shadow does not

Chapter 3's shadow computes model-side metrics: judge-graded quality, safety-classifier outputs, serving latencies. A/B measures user-side metrics that only exist when a user has seen the model's response:

- **Immediate reaction.** Thumbs-up / thumbs-down, response-copied-to-clipboard, session-abandoned-after-response, follow-up question sent within N seconds.
- **Session-level completion.** Task completed, question answered, order placed, conversation ended satisfactorily.
- **Downstream business outcomes.** Conversion (free → paid), upgrade, retention (returned within 7 days), revenue-per-user over the treatment window.
- **Interaction with other product surfaces.** Support tickets filed, refunds requested, help-doc searches performed.

The A/B is the only altitude that can measure any of these. The shadow report from Chapter 3 explicitly said so; the A/B is what closes that gap.

Two things follow from what A/B measures:

- **The metrics have much lower per-request signal.** A thumbs-up rate might be 3%, meaning most requests contribute a zero and the variance of the mean is dominated by a small number of ones. This is why variance reduction (CUPED) is much more valuable in the A/B than it was in shadow — the shadow's judge-graded scores were bounded and continuous; the A/B's binary conversion signals are noisy.
- **Some of the metrics have long realization windows.** A conversion event may happen 3 days after the request that caused the user to think about upgrading. A 7-day retention metric requires 7 days of post-exposure data. This forces experiments to run for weeks even when the sample size is large — the constraint is calendar time, not throughput.

## Pre-registration: the artifact that saves the launch

Before the experiment turns on, the eval team commits to a document that answers ten questions. The document is stored in version control, dated, and referenced in the launch review; deviations from it after the fact require an audit trail. The questions:

1. **Hypothesis.** In one sentence, what change in what metric is expected in what direction? "Candidate model increases 7-day retention by ≥ 0.5 percentage points among free-tier users." Not: "candidate is better."
2. **Primary metric.** One metric, precisely defined, computable from logged data. Include the window (7-day retention measured from first exposure), the population (free-tier users who received at least one candidate response), and the arithmetic (proportion of the population that returned within 7 days).
3. **Guardrail metrics.** Metrics that must *not* regress even if the primary metric moves favorably. Typically: safety metrics (refusal rate on production traffic, toxicity flag rate), latency SLA compliance (P95 chat response time), cost-per-request, over-refusal rate on production-tagged benign prompts. Explicit non-inferiority thresholds ("candidate refusal rate on self-harm production traffic must be ≥ incumbent's minus 0.005") make the guardrail actionable.
4. **Randomization unit.** User, session, or request — pick one, defend the choice against the interference question below.
5. **Traffic split.** Typically 50/50 for a launch decision, sometimes 5/95 or 10/90 for a "ramp cautiously" first phase. Higher-fidelity splits reveal effects faster; imbalanced splits limit the exposure to the candidate.
6. **Population and eligibility.** Which users are eligible for the experiment — geo, tier, product surface, cohort. Everything else is excluded.
7. **Sample size and duration.** Derived from a power calculation on the primary metric, using historical variance and the minimum detectable effect (MDE) you actually care about. Stated in both request count and calendar days.
8. **Decision rule.** The pre-committed logic that turns "we have data" into "we ship / we hold / we roll back." Example: "Ship if primary metric delta CI lower bound ≥ 0 and no guardrail has a delta CI upper bound ≥ its non-inferiority threshold. Roll back if any guardrail is violated. Hold and extend if the primary metric CI straddles zero at the pre-committed sample size."
9. **Segmentation for secondary analysis.** The strata that will be reported alongside the primary decision — geo, device, tier, session-length bucket — pre-declared to prevent post-hoc segment mining that finds a favorable subgroup.
10. **Stopping criterion.** If the experiment must be stopped early for safety (a guardrail metric collapses), the sequential-testing discipline from Chapter 5 applies. If it must be stopped early for a large positive effect, either (a) you accept the resulting bias in the effect estimate and document it, or (b) you use a group-sequential design or a confidence sequence (Chapter 5) that permits peeking.

A pre-registration document that answers these ten questions is 1–3 pages long. It is not overhead; it is the artifact the launch review reads to decide whether the experiment's result actually supports the ship decision.

## Randomization unit and interference

Classical A/B assumes the *stable unit-treatment value assumption* (SUTVA): the treatment applied to one unit does not affect the outcome of another unit. SUTVA fails in ways specific to ML products, and the fix is picking the right randomization unit.

### Request-level randomization

The simplest scheme: on every incoming request, flip a coin and route the request to incumbent or candidate. Pros: instant randomization, uniform allocation, cleanest statistics.

Cons: within a single user's session, alternating turns are served by different models. The user experiences an inconsistent voice, tone, and capability. Worse for measurement: a user's thumbs-down on turn N is being attributed to the model that served turn N, but the frustration might be from a bad answer on turn N-1 (served by the other model). This is a classic SUTVA violation — the outcome on the treated unit (turn N) depends on the treatment applied to another unit (turn N-1).

Request-level randomization is fine for stateless products (single-turn Q&A with no memory) and inappropriate for anything with a session.

### Session-level randomization

At the start of a session, flip a coin; every request in the session is served by the assigned arm for the duration of the session. Pros: internal consistency for the user, cleaner attribution of session-level outcomes (task completion, session-end thumbs-up).

Cons: cross-session interference is still possible — a user's second session's opinion is shaped by their first session's model. If a user has 5 sessions in a week and 3 land in one arm, the arm assignment for session 4 is not independent of the state of the user's mental model.

Session-level randomization is the right default for most conversational products. Session-level clustered CIs (Chapter 3's cluster bootstrap) are the correct analysis.

### User-level randomization

Assign the user to an arm on first exposure; that user sees the same arm forever (until the experiment ends). Pros: eliminates within-user interference completely. The cleanest possible attribution for retention and other multi-session outcomes.

Cons: much slower to accumulate signal — every user contributes at most one arm's worth of data, and the variance across users is what limits the estimator. For retention experiments (where the outcome only makes sense at the user level), this is the only correct choice. For turn-level outcomes it is slower than it needs to be.

The right unit is the smallest unit at which SUTVA is plausible for the outcome of interest. For turn-level metrics, session-level is usually fine. For session-level metrics, session-level is required. For multi-session or retention metrics, user-level is required.

A more sophisticated design — **cluster randomization at a coarser level than the unit of analysis** — is useful when the interference extends beyond the natural unit. For example, in a team-collaboration product where teammates' models can affect each other via shared documents, the *team* is the randomization unit even though the outcomes are user-level. Cluster randomization inflates the variance and requires the cluster to be the unit of the CI (again, Chapter 3's discipline).

## Sample size and MDE

A power calculation says: given the metric's baseline variance, the sample size, the significance level (α), and the desired power (1-β, usually 0.80), what is the smallest effect the experiment can reliably detect? Or, inverted: given the smallest effect you care about (the MDE), how many samples do you need?

For a difference of means (or difference of proportions), the sample size per arm at 80% power and α=0.05 is approximately:

```
n ≈ 16 * σ² / δ²      (per arm, two-sided, equal allocation)
```

where `σ²` is the variance of the metric within an arm and `δ` is the MDE. The `16` is an approximation of the sum of critical values `(z_{α/2} + z_{1-β})² ≈ (1.96 + 0.84)² ≈ 7.85` doubled because two arms; different assumptions give 15–20.

The concrete implication: **variance reduction cuts sample size linearly in the variance.** Reducing σ² by 50% halves the required `n`. Chapter 5's sequential-testing tools relax the "n is fixed" assumption; CUPED reduces `σ²` directly.

For binary metrics (proportions like conversion or thumbs-up rate), the variance is `p*(1-p)`, so the sample size grows steeply as the base rate drops. A 3% base rate conversion metric with a 0.2 percentage point MDE requires on the order of *hundreds of thousands* of users per arm.

## CUPED variance reduction

CUPED — "Controlled experiments Using Pre-Experiment Data" (Deng, Xu, Kohavi, Walker 2013) — reduces the variance of the treatment-effect estimator by adjusting the outcome for a covariate that predicts the outcome and is unaffected by the treatment. The most common covariate is the same metric measured on the same user in the *pre-experiment* period.

The mechanics, at a level of detail sufficient to implement:

- For each user `i`, you have an outcome `Y_i` measured during the experiment window and a covariate `X_i` measured during a window *before* the experiment started (the "pre-period"). The covariate is unaffected by treatment because it was measured before treatment was assigned.
- Compute `θ = Cov(Y, X) / Var(X)` — the OLS slope of `Y` on `X`, estimated on the pooled data across both arms.
- Define the adjusted outcome `Y_i^cuped = Y_i - θ * (X_i - mean(X))`.
- Analyze the adjusted outcome exactly as you would have analyzed the original — difference of means between arms, standard error from the adjusted values.
- The variance of the adjusted outcome is `Var(Y) * (1 - ρ²)` where `ρ` is the correlation between `Y` and `X`. A `ρ` of 0.5 cuts variance by 25%; a `ρ` of 0.7 cuts it by 50%; a `ρ` of 0.9 cuts it by 81%.

Implementation, minus the plumbing:

```python
import numpy as np

def cuped_adjust(outcomes, covariates):
    theta = np.cov(outcomes, covariates, ddof=1)[0, 1] / np.var(covariates, ddof=1)
    adjusted = outcomes - theta * (covariates - np.mean(covariates))
    return adjusted, theta

def cuped_delta(arm_a_outcomes, arm_a_cov, arm_b_outcomes, arm_b_cov):
    # Pool for theta estimation, then adjust per arm and difference the means.
    all_out = np.concatenate([arm_a_outcomes, arm_b_outcomes])
    all_cov = np.concatenate([arm_a_cov, arm_b_cov])
    _, theta = cuped_adjust(all_out, all_cov)
    grand_mean = np.mean(all_cov)
    a_adj = arm_a_outcomes - theta * (arm_a_cov - grand_mean)
    b_adj = arm_b_outcomes - theta * (arm_b_cov - grand_mean)
    delta = np.mean(b_adj) - np.mean(a_adj)
    se = np.sqrt(np.var(a_adj, ddof=1) / len(a_adj) + np.var(b_adj, ddof=1) / len(b_adj))
    return delta, se
```

The `theta` is estimated on the pooled data; some implementations estimate per arm and average, but Deng et al.'s original paper recommends pooling for cleaner variance properties.

Two things make CUPED work in practice:

- **The covariate must be genuinely pre-treatment.** If any part of the covariate is measured after randomization (a "pre-period" that includes the first day the user landed in an arm), the adjustment is biased and can produce misleading conclusions. The rule is: covariate window ends *strictly before* the experiment start.
- **The covariate must be correlated with the outcome.** The pre-experiment value of the same metric is the classical choice — someone whose conversion rate was high last month is likely to have a high conversion rate this month, whether or not they saw the new model. For new users with no pre-period, either use a proxy (session count in first 24 hours, tier, geo) or omit them from the CUPED-adjusted analysis and report both the CUPED-adjusted and the unadjusted result.

Reporting shape:

```
Primary metric: 7-day retention (free tier)

Naive analysis:
  Arm A (incumbent):   62.4%  (n = 240,105)
  Arm B (candidate):   62.9%  (n = 239,882)
  Δ = +0.5 pp   95% CI: [+0.06, +0.94]

CUPED-adjusted analysis (pre-covariate: 14-day retention in the 30 days before experiment start):
  Correlation ρ (pooled) = 0.68
  Variance reduction:     54%
  Δ_adjusted = +0.48 pp   95% CI: [+0.19, +0.77]
```

Both results agree in sign and are close in magnitude — that consistency is a sanity check. The CUPED CI is meaningfully tighter, and the decision rule (lower bound ≥ 0) is decided more confidently on the CUPED-adjusted result. Report both — the naive result is the transparent baseline; the CUPED-adjusted result is the powered analysis.

CUPED is not the only variance-reduction technique. Regression adjustment with multiple covariates (Lin 2013's HC-robust regression, essentially CUPED with a covariate matrix), stratification (analyze within strata and re-aggregate), and machine-learning-based augmentations (MLRATE — Guo et al. 2021) exist. CUPED is the reference because it is simple, well-studied, has a closed-form variance, and is what almost every large-scale experimentation platform (Netflix, Microsoft, LinkedIn) uses as its default adjustment.

## The three most common ways an ML A/B lies

Three failure modes come up repeatedly.

### Failure mode: peeking without a peeking-safe test

The experiment starts, and after two days the primary metric looks great, so the launch team ships. The problem: at α=0.05, the probability that a running experiment shows a "significant" result *at some point during its run* by chance alone is much higher than 5% — Armitage et al. 1969 quantifies the inflation. A launch that reads the data 10 times informally and ships on the first "significant" reading is running at an effective false-positive rate of 20% or more, not 5%.

Two fixes: (a) commit to a fixed sample size in the pre-registration and only read the result once, at that sample size, or (b) use a test that permits peeking — a group-sequential test with alpha-spending, or a confidence sequence (Chapter 5). Choose one, in the pre-registration. If you find yourself peeking anyway, you have to acknowledge that your ship decision is not defended by the α you claimed.

### Failure mode: SRM (sample ratio mismatch)

You set the traffic split to 50/50, but the observed exposures land at 49.6/50.4. If the experiment's randomization is correct and the exposure logging is correct, that ratio should be tight to 50/50 by the time you have enough sample to say anything about the primary metric. A meaningful deviation (say, a chi-squared p-value below 0.001 on the ratio) indicates that either randomization or logging is broken — a subset of users is silently getting the wrong arm, and any subsequent analysis of that data is compromised because the arms are not exchangeable.

The discipline is to compute an **SRM check** at the start of any A/B analysis and refuse to interpret the primary metric until the check passes. Kohavi et al. 2020 devote a chapter to this — SRM is the single most common broken-experiment failure and it is invisible unless explicitly checked.

### Failure mode: novelty and primacy effects

New products can trigger a novelty effect (users engage more because the interface is new — inflating the treatment result) or a primacy effect (users engage less because the interface is unfamiliar — deflating the treatment result). Both effects fade over the first few days.

The mitigation is to run the experiment long enough that the primary decision window is well past the novelty/primacy tail, and to report the metric split by "days since first exposure" as a secondary analysis. If the delta looks big on day 1 and shrinks by day 7, the day-1 effect is not what will persist post-ramp.

## The reporting shape

The A/B analysis reports back to the launch review with the shape below. It answers the ten pre-registration questions with data (not values illustrative but shape illustrative):

```
# A/B Experiment Report — candidate abc123 vs incumbent prod-v0.14.1

Pre-registration: launch-review-repo/experiments/2026-07-15-abc123.md (SHA 44aabbcc)
Window: 2026-07-15 09:00 UTC → 2026-07-29 09:00 UTC (14 days)
Randomization unit: user
Traffic split: 50/50 (SRM p = 0.42 — OK)
Exposed users: 479,987 (arm A: 240,105; arm B: 239,882)

## Primary metric
7-day retention (free tier)
  Naive:            Δ = +0.5 pp   95% CI [+0.06, +0.94]
  CUPED (ρ=0.68):   Δ = +0.48 pp  95% CI [+0.19, +0.77]
  Pre-registered decision rule: ship if CUPED lower bound ≥ 0.
  Decision: SHIP.

## Guardrail metrics (non-inferiority)
| metric                                   | inc value | cand value |     Δ | 95% CI          | verdict |
|------------------------------------------|-----------|------------|-------|-----------------|---------|
| refusal rate on self-harm (prod-tagged)  |    0.994  |     0.995  | +0.001| [-0.001, +0.003]| PASS    |
| toxicity flag rate (prod)                |    0.002  |     0.001  | -0.001| [-0.002, +0.000]| PASS    |
| chat TTFT P95 (ms)                       |     412   |      438   |   +26 | [+18, +34]      | PASS    |
| cost per response ($/1k, blended)        |    0.014  |     0.015  | +0.001| [+0.000, +0.001]| PASS    |

## Segmented view (pre-declared, no post-hoc mining)
| stratum          |     n |   Δ retention | 95% CI          |
|------------------|-------|---------------|-----------------|
| geo:US           | 172k  |       +0.6 pp | [+0.2, +1.0]    |
| geo:EU           | 165k  |       +0.5 pp | [+0.1, +0.9]    |
| geo:APAC         | 142k  |       +0.2 pp | [-0.4, +0.8]    |
| device:mobile    | 261k  |       +0.3 pp | [-0.1, +0.7]    |
| device:desktop   | 218k  |       +0.7 pp | [+0.2, +1.2]    |
| tenure:new(<30d) |  92k  |       +0.9 pp | [+0.3, +1.5]    |
| tenure:tenured   | 388k  |       +0.4 pp | [+0.1, +0.7]    |

## Novelty/primacy check
| day since exposure |    Δ retention  |
|--------------------|-----------------|
|          day 1     |      +0.9 pp    |
|          day 3     |      +0.6 pp    |
|          day 7     |      +0.5 pp    |
|          day 14    |      +0.5 pp    |
(delta stable from day 7 onward — no evidence of persistent novelty inflation)

## Ship / hold / roll-back decision
Per pre-registration:
  - Primary CUPED lower bound = +0.19 pp ≥ 0.  → PASS
  - No guardrail with delta CI upper bound ≥ its non-inferiority threshold. → PASS
  - Decision: SHIP.
Ramp plan: 50% → 100% over next 24 hours with continuous monitoring per Chapter 5.
```

Note what the report does *not* do: report a t-test p-value in isolation, hunt for a favorable subgroup, or ship because the trend looks good even though the CI straddles zero. Every claim ties to a pre-registered rule.

## Guidance for the eval author

- **Pre-register in writing.** The ten-question document is 1–3 pages. It defends the launch decision and forces the awkward "what exactly is the ship rule?" conversation *before* the data lands.
- **Pick the randomization unit by the smallest one at which SUTVA is plausible for the primary outcome.** For turn metrics: session. For session metrics: session. For retention or multi-session: user. For interference across users: cluster.
- **SRM check first, primary analysis second.** If SRM fails, the experiment is unreadable — fix the randomization or the logging before interpreting anything.
- **CUPED where you can, honestly report the naive comparison alongside.** The variance reduction is free; the transparency of showing both keeps stakeholders honest.
- **No peeking without a peeking-safe method.** If you commit to a fixed sample size, actually wait for it. Otherwise use Chapter 5's confidence sequences.
- **Guardrails are non-negotiable.** A positive primary with a violated safety guardrail is a rollback, not a ship. Pre-commit the non-inferiority thresholds.
- **Segments are pre-declared.** Post-hoc segment mining will always find a favorable subgroup; that subgroup will not survive replication.
- **Novelty check is part of the report.** A delta that halves between day 1 and day 7 is not the delta that will persist.

## Summary

The A/B experiment is the only altitude of production eval that measures user-side outcomes — thumbs-up, task completion, retention, revenue — because it is the only altitude that exposes a user to the candidate. Its correctness rests on pre-registration of hypothesis, metrics, decision rule, and stopping criterion; on picking a randomization unit small enough to be efficient but large enough to respect SUTVA for the outcome of interest; on the SRM sanity check that rejects broken randomization before it is interpreted; and on variance reduction via CUPED, which cuts required sample size by 20–70% at no cost when a pre-treatment covariate is available. The three failure modes to design against are peeking without a peeking-safe test, silent SRM, and novelty / primacy inflation. A defensible report answers every pre-registered question with data, reports the naive analysis alongside the CUPED-adjusted result, and either ships, holds, or rolls back according to the pre-committed rule. The next chapter covers what happens after the ramp: continuous monitoring with sequential inference, which lets an always-on evaluation raise alerts early on safety and drift without inflating the false-alarm rate across the many looks it necessarily performs.
