# What an Eval Actually Measures

An eval is a measurement instrument. It takes a system (a model, a prompt, a pipeline) and returns a number that stands in for something we care about — quality, safety, capability, cost. Everything else in this module is machinery for making that measurement trustworthy: does the number mean what we think it means, and how sure are we of the number itself?

This chapter frames the rest of the module. Later chapters cover the individual tools; here we lay out the two questions every eval report has to answer and the two ways almost every eval report gets those questions wrong.

## The two questions every eval answers

Every eval that gets used to make a decision — ship this model, block that release, pick judge A over judge B — implicitly claims:

1. **The score corresponds to the thing we care about.** If MMLU goes up by two points, the model *actually* knows more. If the LLM-judge win-rate goes up, users would *actually* prefer that model in production. This is the **validity** question.
2. **The score is not noise.** If we ran the eval again on a fresh sample, or with a different random seed, we would see roughly the same number. This is the **estimation** question.

Both are ways of asking "what would we see if we did this over?" — a validity failure means we'd see the same number but it wouldn't mean what we said; an estimation failure means the number itself would jump around.

A model card that reports `MMLU: 71.3` speaks to neither. To turn that headline into a decision you need a validity story (what does 71.3 imply, for whom, on what deployment surface) and an estimation story (how tight is the interval, and is it tight enough to distinguish 71.3 from the baseline's 69.8).

## Why single-number eval reports are the default failure mode

Eval reports overwhelmingly ship point estimates without intervals. A leaderboard row is a scalar. A "we improved MMLU by 0.4 points" tweet is a scalar. This is the default because point estimates are cheap to compute, cheap to read, and easy to sort.

They are also the default failure mode. Two examples of what a point estimate hides:

- **Small dataset, wide interval.** On HumanEval (164 problems) with pass@1, a 5-percentage-point improvement is well inside the 95% Wilson interval of the baseline. A "5 points on HumanEval" claim without an interval is not evidence of an improvement. Chapter 3 shows the arithmetic.
- **Paired scores, unpaired reporting.** Two models are usually run on the *same* prompts. The correct comparison is per-item — model A wins on some items, model B on others — and the effective sample size for the difference is the number of *disagreements*, not the number of items. Reporting two independent CIs and eyeballing overlap systematically underpowers the comparison. Chapter 5 covers McNemar and the paired bootstrap.

The single-number report also masks the harder problem: even if the number is correctly estimated, does it measure the right thing? A model with a 92% factuality score on TriviaQA has been measured on TriviaQA, not on "factuality." That gap is a validity gap, not an estimation gap. Chapter 2 covers the three validity frames used to talk about it.

## Terminology we will use consistently

The stats literature and the ML literature use slightly different words for the same objects. To keep the chapters short we fix vocabulary now.

- **Point estimate.** The scalar the eval reports. Written `p̂` for a proportion (accuracy, pass rate, win rate) and `θ̂` for a more general statistic (F1, ROUGE, ELO).
- **Standard error.** The estimated standard deviation of the point estimate across hypothetical re-runs on fresh samples of the same size.
- **Confidence interval (CI).** An interval `[L, U]` computed from the sample such that, under repeated sampling, `[L, U]` covers the true parameter with the stated probability (e.g. 95%). CIs are properties of the *procedure*, not of the specific interval you got.
- **Non-asymptotic.** Guarantees that hold at the sample size you actually have, not "in the limit as n → ∞." The Wilson interval is non-asymptotic in a useful sense (its coverage is well-behaved down to `n ≈ 10`); the Wald interval is asymptotic and misbehaves at small n and extreme p̂.
- **Paired data.** Two systems evaluated on the same items, giving per-item outcomes `(a_i, b_i)`. Almost every model-vs-model comparison you will run is paired.
- **Slice.** A subset of the eval set — by language, by input length, by domain, by demographic group. Slice reporting is where multiple-comparison problems appear (Chapter 6).

## What "the truth" means in an eval

A confidence interval covers "the true parameter." What is that true parameter, concretely?

For a fixed eval set of `n` items and a deterministic scorer, running the model on all `n` items gives you the exact per-item outcomes with no sampling uncertainty at all. The uncertainty enters because the `n` items are themselves a sample from some larger population you care about — production traffic, all English arithmetic problems, all safety-relevant prompts. The CI is a statement about that population parameter, given that our items are a sample from it.

If you treat the eval set as the population (e.g. "our benchmark IS TriviaQA, we do not care about generalization"), then there is no sampling uncertainty and a CI is not the right object; you would instead reason about scorer noise, prompt-format sensitivity, and seed variance. Most real eval questions do involve generalization, so most real eval reports need a CI. Where they do not (e.g. an internal integration test where the eval set *is* the target), say so explicitly.

Two more sources of variability the CI does not, on its own, address:

- **Scorer noise.** If the scorer is an LLM judge or a human panel, re-running scoring on the same outputs will not give the same scores. Chapter 4 shows how to bootstrap over the joint (item, scorer-run) resample when this matters.
- **Prompt / decoding sensitivity.** The same model on the same eval with a different chat template, a different system prompt, or a different sampling temperature can move several points on a leaderboard. This is a construct-validity concern (Chapter 2) more than a CI concern.

## Roadmap

The rest of this module follows the two questions above.

- **Validity (Chapter 2).** Construct, internal, and external validity — the three frames for asking "does the number mean what we said."
- **Estimation for a single system (Chapters 3–4).** Wilson intervals for proportions, percentile and paired bootstraps for anything else.
- **Comparing two systems (Chapter 5).** McNemar's test, the paired bootstrap, the sign test, and why power at small N is the thing that actually decides whether you can call a change.
- **Comparing many systems or slices (Chapter 6).** Benjamini–Hochberg FDR control, when to apply it, and — the more common mistake — what *not* to correct for.
- **Reading an eval report (Chapter 7).** Putting the previous six chapters to work on a report someone else wrote.

## Summary

An eval is a measurement instrument, and a useful eval report has to answer both "does the number mean what we think" (validity) and "how sure are we of the number" (estimation). Single-number reports skip both by default. The rest of this module is the machinery — validity frames, non-asymptotic CIs, paired tests, FDR control — for making an eval report you would actually trust to gate a release.
