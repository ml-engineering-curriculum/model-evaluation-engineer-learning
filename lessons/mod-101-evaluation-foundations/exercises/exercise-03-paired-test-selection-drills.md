# exercise-03: Paired Test Selection Drills

**Estimated effort:** 2 hours

## Objective

Given a set of realistic eval-comparison scenarios, pick the correct paired-comparison test for each, defend the choice against the two most plausible alternatives, and — for at least half the scenarios — compute a power estimate that says whether the eval as described is actually adequate to detect the effect the report is claiming.

The goal is diagnostic fluency, not code volume. Most of the work is in the reasoning.

## Prerequisites

- Chapters 03, 04, and 05 of this module.
- `python` with `statsmodels` and `numpy` for the power calculations.

## Requirements

### Part A — the scenario cards (do all 8)

For each scenario below, produce a short structured response:

- **Chosen test:** one of `McNemar (exact)`, `McNemar (χ²)`, `paired bootstrap`, `sign test`, `Wilcoxon signed-rank`, or `other` (specify).
- **Why this and not two named alternatives:** 2–4 sentences comparing to the two most plausible other choices for this scenario.
- **Effective sample size:** the `n_eff` that actually determines the test's power (discordant count for McNemar, non-tie count for sign test, cluster count for a clustered design).
- **Verdict:** is the test likely to have adequate power (≥ 80%) for the effect the report is claiming? Use the MDE table in Chapter 5 or a `statsmodels` power call to justify.

Scenarios:

1. Two classifiers evaluated on the same 200-image test set. Each image is labeled 0/1; both classifiers produce 0/1 predictions. Classifier A is right 84% of the time, Classifier B is right 87% of the time. They agree on 170 images, disagree on 30 (A right / B wrong on 8; B right / A wrong on 22).
2. Two LLMs run on the same 400-prompt helpfulness benchmark. An LLM judge scores each prompt as "A better," "B better," or "tie." Result: A wins 130, B wins 190, ties 80.
3. Two LLMs run on the same 100-item long-form summarization eval. A human panel gives each summary a Likert score in {1, 2, 3, 4, 5}. Mean A = 3.4, Mean B = 3.6, standard deviation of the paired difference = 0.8.
4. A single LLM evaluated on the same 500-example code-generation benchmark, once with `temperature=0` and once with `temperature=0.7`. Pass@1: `temp=0` = 41.2%, `temp=0.7` = 44.4%. Same 500 problems in both runs.
5. Two models compared on a 300-item long-form QA eval with an LLM judge. The judge is stochastic; you ran the judge **5 times** per (item, model). B wins per-item (majority of judge runs) 60% of the time. Effective independence unclear.
6. Two models compared on a 50-item internal benchmark. Model A is right on 40, Model B is right on 42. They agree on 44 items and disagree on 6 (A right / B wrong on 1; B right / A wrong on 5).
7. A model regression check: today's model vs. last week's model on a 1000-example canary set, both giving binary correctness. Discordant cells: `b = 8`, `c = 22`. The report claims "significant regression."
8. Two RAG systems compared on the same 200 user queries. Each query yields a per-query pass/fail. The queries are grouped into 40 user sessions of 5 queries each; queries within a session share context and are not independent.

### Part B — one worked-in-full computation

Pick any two of the scenarios above and, for each, produce a full worked computation:

- Compute the test statistic and p-value by hand or with a documented library call. Show the intermediate arithmetic (contingency table, discordant count, `p_1`, `p_2`, etc.).
- Compute the 95% CI on the effect (paired-bootstrap CI on the difference for continuous metrics; Wilson or McNemar-derived CI for binary metrics).
- State the minimum-detectable-effect at 80% power for the effective sample size, using the MDE table in Chapter 5 or a `statsmodels.stats.power` call.
- Write a two-sentence verdict for a technical reviewer.

### Part C — the "wrong test" postmortem

For **one** of the eight scenarios, pick a common wrong-test choice a beginner might make (e.g. two-sample t-test on scenario 1, unpaired proportion test on scenario 6, bootstrap on the marginal accuracies with no pairing on scenario 4). Compute what that wrong choice would say, explain the mechanism of the error, and quantify how much it would inflate or deflate the significance verdict.

## Starter guidance

- The Chapter 5 heuristic table is the fast path — walk down it. If it doesn't match, that is a signal you have identified an edge case (clustering, stochastic scorer, multi-way outcomes) worth writing about.
- For scenario 8 (clustered), the two candidate approaches are: (a) cluster-bootstrap over sessions, (b) session-level statistic (average per session) treated as the item. Either is defensible; say which and why.
- For scenario 5 (stochastic judge), the two candidate approaches are: (a) aggregate to a single per-item outcome (majority vote across judge runs) and use McNemar, (b) hierarchical bootstrap that resamples items and then judge runs. Discuss the trade-off.
- The MDE table gives you a fast normal-approximation power check. For scenarios with `b + c ≤ 30`, use the exact binomial power computation (`from scipy.stats import binom` and integrate the rejection region) rather than the normal approximation.
- `statsmodels.stats.contingency_tables.mcnemar(table, exact=True)` returns both the test statistic and the p-value. For paired bootstrap use the function you wrote in exercise-02 or `scipy.stats.bootstrap`.

## Acceptance criteria

Your submission is acceptable if:

- All 8 scenario cards have the four fields (test, reasoning, `n_eff`, power verdict).
- The two worked computations in Part B include the arithmetic (not just "p = 0.03") and state the MDE.
- The Part C postmortem quantifies the error (e.g. "the wrong test says `p = 0.008`; the correct test says `p = 0.11`; the wrong test inflates significance because it treats paired outcomes as independent") rather than describing it in words alone.
- Scenario 8 identifies clustering as the load-bearing issue and picks one of the two defensible responses.
- Scenario 5 identifies judge noise as the load-bearing issue and picks one of the two defensible responses.

## Stretch goals

- Write a small `pick_paired_test(design_dict) -> str` helper that encodes the Chapter 5 decision table plus your Part A reasoning, and run it against all 8 scenarios as a regression test.
- For scenario 3 (Likert with `sd = 0.8`), compute the paired-`t` p-value, the Wilcoxon signed-rank p-value, and the paired-bootstrap p-value. Compare all three and explain the differences.
- Reproduce the MDE table in Chapter 5 using an **exact** binomial power calculation and note where the normal approximation diverges materially from the exact result (typically at `n ≤ 30` and effects near the boundary).
