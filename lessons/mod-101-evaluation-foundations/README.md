# mod-101-evaluation-foundations: Evaluation Foundations: Validity, Estimation, and the Math of Measurement

**Estimated effort:** 12 hours

Foundations for every downstream module in the track. This module builds the two questions every eval has to answer — *does the number measure what we say it does* (validity) and *how sure are we of the number* (estimation) — and gives you the specific tools to answer both: Wilson intervals for proportions, bootstrap CIs for everything else, paired tests for comparisons, and Benjamini–Hochberg FDR control for multi-slice / multi-metric reports. It closes with the reading protocol you will apply to every eval report you encounter afterwards.

## Learning objectives

- Distinguish construct, internal, and external validity and identify the dominant validity threat for a given eval design.
- Produce point estimates with non-asymptotic confidence intervals (Wilson, paired bootstrap, percentile bootstrap) for accuracy, F1, and pairwise win-rates.
- Pick the correct paired-comparison test (McNemar, paired bootstrap, sign test) and explain why power matters at small N.
- Apply Benjamini–Hochberg FDR control when reporting many slices or many metrics, and explain what *not* to correct for.
- Read an eval report and locate the validity, sampling, statistical-power, and contamination weaknesses.

## Lecture chapters

1. [`01-what-an-eval-actually-measures.md`](01-what-an-eval-actually-measures.md) — framing: eval as measurement, the validity and estimation questions, why single-number reports fail.
2. [`02-validity-construct-internal-external.md`](02-validity-construct-internal-external.md) — construct / internal / external validity, dominant-threat heuristic table, worked example.
3. [`03-point-estimates-and-wilson-intervals.md`](03-point-estimates-and-wilson-intervals.md) — proportions, why Wald is wrong at eval sample sizes, the Wilson score interval.
4. [`04-bootstrap-confidence-intervals.md`](04-bootstrap-confidence-intervals.md) — percentile bootstrap for F1/averages, paired bootstrap for win-rate differences, clustering and scorer-noise gotchas.
5. [`05-paired-comparison-tests.md`](05-paired-comparison-tests.md) — McNemar, paired bootstrap test, sign test, picking between them, and the MDE table for power planning.
6. [`06-multiple-comparisons-and-bh-fdr.md`](06-multiple-comparisons-and-bh-fdr.md) — Benjamini–Hochberg procedure, FWER vs. FDR, when to correct and what not to correct.
7. [`07-reading-eval-reports-critically.md`](07-reading-eval-reports-critically.md) — the four-question protocol, a 12-item checklist, worked walkthroughs for judge-based, benchmark, and A/B reports.

## Exercises

Five hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after finishing the chapters it depends on.

- [`exercise-01-construct-validity-audit.md`](exercises/exercise-01-construct-validity-audit.md) — construct validity audit of a published eval design.
- [`exercise-02-bootstrap-ci-implementation.md`](exercises/exercise-02-bootstrap-ci-implementation.md) — implement paired and percentile bootstrap CIs from scratch, validate against a library.
- [`exercise-03-paired-test-selection-drills.md`](exercises/exercise-03-paired-test-selection-drills.md) — pick the right paired test for a set of scenario cards and defend the choice.
- [`exercise-04-fdr-controlled-slice-reporting.md`](exercises/exercise-04-fdr-controlled-slice-reporting.md) — take a multi-slice results table, apply BH, and produce a corrected report.
- [`exercise-05-eval-report-red-flag-review.md`](exercises/exercise-05-eval-report-red-flag-review.md) — apply the 12-item checklist to a supplied eval report and write a reviewer memo.

Reference solutions live in the paired [`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for the primary references — Wilson 1927, Agresti & Coull 1998, Efron 1979, Dietterich 1998, Benjamini & Hochberg 1995, Cronbach & Meehl 1955, Recht et al. 2019, and the standard book-length treatments.
