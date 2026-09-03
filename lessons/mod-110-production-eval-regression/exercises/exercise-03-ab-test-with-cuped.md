# exercise-03: A/B Experiment With CUPED Variance Reduction

**Estimated effort:** 3 hours

## Objective

Design and simulate a controlled A/B experiment for a model release the way Chapter 4 prescribes. Author a pre-registration document that answers the ten questions, generate a synthetic user-level dataset that exercises the primary metric, guardrails, and pre-treatment covariates, run the analysis with both a naive difference-in-means and a CUPED-adjusted estimator, apply an SRM check and a novelty / primacy trace, and produce a launch-review report that ends in a `SHIP / HOLD / ROLL-BACK` decision reached against the pre-registered rules. The exercise is not about running a live A/B (that requires production infrastructure and real users); it is about building the analysis pipeline and the reporting shape so that when a real A/B is available, the pipeline reads its result honestly.

## Prerequisites

- mod-110 Chapter 4 (controlled A/B experiments with CUPED and pre-registered hypotheses), Chapter 3 (paired analysis, cluster CIs).
- mod-101 Chapter 2 (bootstrap CIs), Chapter 3 (paired comparisons), Chapter 4 (family-wise error across many metrics).
- mod-105 for the judge-graded secondary metrics if you extend beyond binary user-side outcomes.
- Kohavi, Tang & Xu 2020, *Trustworthy Online Controlled Experiments*, §§ 3, 8, 22 — pre-registration, SRM, CUPED — read alongside the chapter.
- Python 3.11+ with `numpy`, `scipy`, `pandas`, and (optional) `statsmodels` for the SRM chi-squared check.

## Datasets

You need a user-level dataset with (a) a pre-experiment period providing the CUPED covariate and (b) an experiment window providing the outcome. Two sourcing options — pick one:

- **Synthetic user-level generator (recommended).** Ship `abtest/simulate.py` that draws `N ≥ 200,000` users with:
  - `user_id`.
  - `tenure_days`, `tier ∈ {"free", "paid"}`, `geo ∈ {"US", "EU", "APAC"}`, `device ∈ {"mobile", "desktop"}` — the pre-declared segmentation axes.
  - A **pre-covariate** `X_i` (e.g., 14-day retention in the 30 days before experiment start), drawn from a distribution whose correlation with the outcome you control via a parameter `rho ∈ [0.0, 0.9]`.
  - An **arm assignment** `arm ∈ {"control", "treatment"}` with a nominal 50/50 split. Include a switch that induces a small SRM (e.g., 49.5/50.5) so the SRM check has a real signal to catch.
  - A **primary outcome** `Y_i` (binary 7-day retention) with a true treatment effect `tau` (default `+0.005`, i.e. 0.5 percentage points).
  - **Guardrail outcomes**: refusal rate on a synthetic safety-tagged sub-population, TTFT P95, cost per response — with treatment effects you can dial to zero, favorable, or adverse.
  - A **daily exposure trace** `Y_{i,d}` for `d ∈ [1, 14]` that lets you compute the day-since-exposure novelty check.
- **Public-data adaptation.** Use a public log-style dataset (e.g., MovieLens as a proxy for repeated-visit behavior) with a synthetic arm assignment and a synthetic treatment effect. Document the mapping.

Every dataset load records `(generator_seed, n_users, rho, tau, split, srm_bias, guardrail_effects)` in a manifest.

## Requirements

### Part A — the pre-registration

Ship `PRE_REGISTRATION.md` *before* writing the analysis. Answer the ten questions from Chapter 4:

1. Hypothesis (one sentence, direction and magnitude).
2. Primary metric (precisely defined: window, population, arithmetic).
3. Guardrail metrics (with non-inferiority thresholds).
4. Randomization unit (user, session, or request — justify against SUTVA for the primary outcome).
5. Traffic split.
6. Population and eligibility.
7. Sample size and duration (with a power calculation, `α = 0.05`, `1-β = 0.80`).
8. Decision rule (`SHIP / HOLD / ROLL-BACK` logic, explicit).
9. Segmentation for secondary analysis (pre-declared strata — geo, device, tier, tenure).
10. Stopping criterion (fixed-`n` or group-sequential — commit to one).

Store the document in git, commit it before the analysis code runs, and make the analysis script fail if the pre-registration file is missing or has been modified after the analysis started.

### Part B — the SRM check

Implement `srm_check(arm_counts, expected_split)` that returns a chi-squared p-value on the observed vs. expected arm counts. The analysis pipeline must call this *first* and refuse to interpret the primary metric if `p < 0.001` — per Kohavi et al. 2020's threshold. Include a unit test that verifies the check fires on your simulator's SRM-biased mode and passes on the balanced mode.

### Part C — the naive and CUPED estimators

Implement two side-by-side estimators in `abtest/estimators.py`:

- `naive_delta(y_control, y_treatment)` — difference of means, standard error from the pooled variance, 95% CI.
- `cuped_delta(y_control, y_treatment, x_control, x_treatment)` — the Chapter 4 recipe: estimate `θ` on pooled data, form `Y^cuped = Y - θ*(X - mean(X))`, difference the arm means on the adjusted outcome, standard error from the adjusted variance. Return `(delta, se, ci_low, ci_high, theta, rho_hat, variance_reduction_pct)`.

Include unit tests that verify:

- On synthetic data with `rho ≈ 0.7`, CUPED variance reduction is ≥ 40%.
- On synthetic data with `rho = 0`, CUPED and naive give the same delta and (almost) the same SE.
- The CUPED estimator is unbiased under the null (`tau = 0`).

### Part D — the guardrails

Every guardrail is a non-inferiority check with a pre-committed margin. Ship `guardrail_check(inc_value, cand_value, se, margin, direction)`:

- `direction ∈ {"lower_is_better", "higher_is_better"}`.
- Returns `PASS` iff the CI of the delta does not cross the non-inferiority margin.

Enforce that any guardrail failure blocks the `SHIP` verdict regardless of the primary result. This is Chapter 4's rule: a positive primary with a violated guardrail is a `ROLL-BACK`, not a ship.

### Part E — the novelty / primacy trace

Compute the delta separately per `day_since_exposure ∈ {1, 3, 7, 14}` and report it as a table. Flag the case where `day_1_delta / day_14_delta > 2` as a suspected novelty inflation and require the report to name the finding.

### Part F — the segmented view

Compute the CUPED-adjusted delta per pre-declared stratum (geo, device, tier, tenure). Post-hoc segmentation is disallowed — segments must appear in `PRE_REGISTRATION.md` or the report refuses to include them. Report each stratum's `(n, delta, 95% CI)`.

### Part G — the report

`python -m abtest.report exp.jsonl --pre PRE_REGISTRATION.md --out report.md` produces the shape from Chapter 4:

- Header (pre-registration SHA, window, unit, split, SRM p-value).
- Primary metric block with both naive and CUPED results and the decision rule outcome.
- Guardrails table with verdicts.
- Segmented view (pre-declared strata only).
- Novelty check table.
- Final `SHIP / HOLD / ROLL-BACK` block that names each pre-registered rule and its status.

## Starter guidance

- **Author the pre-registration before touching the estimator.** The hardest part of this exercise is committing to a decision rule before seeing the numbers. Do it first, in a separate git commit.
- **CUPED needs a strictly-pre-treatment covariate.** If your simulator's covariate uses any data from after `experiment_start`, the CUPED estimator is biased and the exercise's variance-reduction claim is a lie. Enforce this in the loader.
- **SRM check before anything else.** The single-most-common broken-experiment failure. If your SRM check does not fire on your simulator's biased mode, the check is broken; do not proceed until the unit test passes.
- **The naive result stays in the report.** Chapter 4 was explicit: report both. The naive is the transparent sanity check; CUPED is the powered analysis. If they disagree in sign, that is itself a finding.
- **A guardrail failure is a `ROLL-BACK`, not a warning.** The pre-registration commits the org to this; the report must implement it.
- **No post-hoc segments.** If a segment does not appear in `PRE_REGISTRATION.md`, the report does not report it. This is where post-hoc mining lives; kill it at the code level.
- **Do not print user-level rows in the report.** Aggregates and CIs only. Individual-user data lives in a restricted-permissions log directory.

## Acceptance criteria

- `PRE_REGISTRATION.md` exists, answers all ten Chapter 4 questions, and was committed before the analysis code (a `git log` inspection is the honest check).
- The SRM check unit test passes: fires on the biased mode, passes on the balanced mode.
- The naive and CUPED estimators are implemented and pass their three unit tests (correct variance reduction on high `rho`, no-op on `rho = 0`, unbiased under null).
- On the default simulator (`tau = +0.005`, `rho = 0.68`, `n = 200,000` per arm), the CUPED CI is at least 30% narrower than the naive CI, and both agree in sign.
- Guardrail checks are implemented; a synthetic run with an adverse safety guardrail produces `ROLL-BACK` even when the primary is positive.
- The report's day-1-vs-day-14 novelty table is populated; a simulator run with a strong novelty effect (`day_1 delta = 3 * day_14 delta`) is flagged in the report.
- Segmentation strata match the pre-registration exactly; a segment not in the pre-registration cannot be added post-hoc without modifying the pre-registration file (and thus its SHA).
- The pipeline is reproducible under a seed — two runs of the same simulator config produce byte-identical CIs and the same final verdict.

## Stretch goals

- **Group-sequential design with O'Brien–Fleming alpha spending.** Add an optional stopping-criterion mode that plans four interim looks (25/50/75/100% of `n_max`) with O'Brien–Fleming critical values via `statsmodels.stats.proportion.samplesize_confint_proportion` scaffolding or the `groupsequential` helpers in `scipy` / an internal library. Re-run the simulator with a true effect and show that the experiment stops earlier than the fixed-`n` design.
- **Regression adjustment beyond CUPED.** Implement Lin 2013's HC-robust regression with a covariate matrix (tier, geo, tenure, pre-covariate) and compare its variance reduction against single-covariate CUPED on the same data.
- **MLRATE (Guo et al. 2021) as a variance-reduction ceiling.** Train a small model (`sklearn` gradient boosting) on the pooled control-arm data to predict `Y` from covariates, then use its predictions as the CUPED-style covariate. Compare variance reduction against the single-covariate baseline; report where MLRATE helps and where it does not.
- **Session-level randomization variant.** Rewrite the simulator to emit session-level exposures (each user has `k ~ Poisson(3)` sessions per week), randomize at the session level, and re-analyze with a session-level cluster bootstrap. Show that the primary CI is wider than the user-randomized version and discuss the trade-off.
- **Wire into an observability platform.** Attach the per-user CUPED-adjusted outcomes as evaluations on span records in Phoenix / Langfuse / Weave (Chapter 6). The report becomes a queryable artifact rather than a one-off markdown file.
