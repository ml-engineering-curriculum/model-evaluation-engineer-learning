# mod-103-classical-ml-eval-depth: Classical ML Evaluation Depth: Calibration, Slicing, and Fairness

**Estimated effort:** 14 hours

Classical ML evaluation is where the discipline the previous modules built — validity, estimation, and benchmark hygiene — starts producing the artifacts a product team actually acts on: a per-slice metrics table, a calibration diagnosis, a chosen operating point with an explicit cost story, a fairness measurement, and a regression/ranking report with subgroup breakdowns. Every LLM-eval, judge-eval, and agent-eval module that follows reuses these primitives. If you skip them, later modules become "prompt engineering with a validation set" instead of measurement.

This module is deliberately quantitative. Each chapter starts from a specific product-side question ("what score do I ship?", "how sure is the classifier when it says 0.9?", "did we hurt one segment more than another?") and shows the estimator, the report format, and the failure mode.

## Learning objectives

- Build a per-slice / per-segment metrics report and justify the slicing axes from product / risk surface.
- Measure calibration with ECE and reliability diagrams, then recalibrate with Platt scaling, isotonic regression, and temperature scaling — and decide which.
- Pick an operating point from a PR / ROC curve under explicit cost asymmetry and report it correctly.
- Run fairness measurement (demographic parity, equalized odds, equal opportunity) with Fairlearn / Aequitas and reason about the impossibility results.
- Evaluate regression and ranking models (RMSE, MAE, MAPE, R², NDCG, MAP) with subgroup breakdowns.

## Lecture chapters

1. [`01-slicing-axes-and-per-slice-metrics.md`](01-slicing-axes-and-per-slice-metrics.md) — why aggregate metrics hide failures, how to pick slicing axes from the product and risk surface, Simpson's paradox, minimum sample size per slice, the per-slice report artifact.
2. [`02-calibration-and-reliability-diagrams.md`](02-calibration-and-reliability-diagrams.md) — what calibration means, the reliability diagram, expected and maximum calibration error, fixed vs. adaptive binning, and the Brier-score decomposition.
3. [`03-recalibration-platt-isotonic-temperature.md`](03-recalibration-platt-isotonic-temperature.md) — Platt scaling, isotonic regression, temperature scaling — how each fits, when each is appropriate, and how to validate on held-out data.
4. [`04-operating-point-selection-under-cost-asymmetry.md`](04-operating-point-selection-under-cost-asymmetry.md) — PR and ROC curves, the cost matrix, expected-utility maximization, Neyman–Pearson constrained selection, and how to report the operating point defensibly.
5. [`05-fairness-metrics-parity-odds-opportunity.md`](05-fairness-metrics-parity-odds-opportunity.md) — demographic parity, equalized odds, equal opportunity, predictive-parity, calibration-within-group; the four-fifths rule and disparate impact in US law; how to define the sensitive attribute.
6. [`06-fairness-impossibility-and-tooling.md`](06-fairness-impossibility-and-tooling.md) — the Chouldechova / Kleinberg–Mullainathan–Raghavan impossibility results, using Fairlearn and Aequitas end-to-end, and the difference between measurement and mitigation.
7. [`07-regression-and-ranking-evaluation.md`](07-regression-and-ranking-evaluation.md) — RMSE, MAE, MAPE, R², sMAPE and quantile loss for regression; MAP, NDCG, MRR, Recall@k for ranking; subgroup breakdowns and pitfalls.

## Exercises

Five hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after finishing the chapters it depends on.

- [`exercise-01-per-slice-metrics-dashboard.md`](exercises/exercise-01-per-slice-metrics-dashboard.md) — construct the per-slice metrics table for a real tabular classifier and justify the axes with a written risk register.
- [`exercise-02-calibration-and-recalibration.md`](exercises/exercise-02-calibration-and-recalibration.md) — measure ECE and produce a reliability diagram, then recalibrate with all three methods and pick the winner.
- [`exercise-03-operating-point-selection-under-cost-asymmetry.md`](exercises/exercise-03-operating-point-selection-under-cost-asymmetry.md) — take a supplied cost matrix and business-side constraint and pick / defend an operating point end to end.
- [`exercise-04-fairness-measurement-audit.md`](exercises/exercise-04-fairness-measurement-audit.md) — run Fairlearn and Aequitas against a classifier and write a fairness measurement report that reasons about the impossibility trade-off.
- [`exercise-05-regression-and-ranking-eval-suite.md`](exercises/exercise-05-regression-and-ranking-eval-suite.md) — build a reusable regression + ranking eval harness with subgroup breakdowns for a supplied model.

Reference solutions live in the paired [`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for primary references — Niculescu-Mizil & Caruana 2005, Guo et al. 2017 on temperature scaling, Platt 1999, Zadrozny & Elkan 2002, Elkan 2001 on cost-sensitive learning, Hardt et al. 2016 on equalized odds, Chouldechova 2017 and Kleinberg et al. 2017 on impossibility, the Fairlearn and Aequitas project documentation, Järvelin & Kekäläinen 2002 on NDCG, and the scikit-learn evaluation documentation.
