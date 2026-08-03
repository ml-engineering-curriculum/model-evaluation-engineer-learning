# Why a Model Judges a Model (And What the Judge Actually Is)

The previous module put you inside the four harnesses the field runs benchmarks through. Most of what those harnesses score is either a log-likelihood over a fixed continuation or a normalized string match against a reference. That works for MMLU and GSM8K. It stops working the moment the eval question is "did the assistant write a good summary," "did the reply actually address the user's problem," or "which of these two answers is better for a coding task." There is no reference string to match. There often is not even a single right answer. This is the failure mode that LLM-as-judge platforms exist to fill: use a second capable model as the scoring function, prompt it with a rubric, and treat its verdict as the metric.

This chapter is about what that decision actually costs you. A judge model is a *measurement instrument*, and instruments have systematic error. The rest of the module is largely about identifying those errors — position bias, length bias, self-preference, rubric ambiguity — and about calibrating a judge against humans so you know whether the number it produces is trustworthy. Before any of that, though, you need to be clear about which of two very different measurement modes you are asking the judge to operate in.

## The two modes: absolute and pairwise

Almost every LLM-as-judge setup you will encounter is a variant of one of two shapes.

**Absolute (single-answer) scoring.** The judge sees one prompt-response pair at a time and returns a score on a fixed scale, or a categorical label, according to a rubric. "Rate this summary from 1 to 5 on faithfulness." "Label this reply as safe / borderline / unsafe." "Did the model follow all three formatting constraints? Yes / No." The score for a system is the mean (or fraction) of per-item judgments. This is the natural shape when you want a *point-in-time metric* — a number you can put on a dashboard, watch across releases, or gate a deploy on. It is also the shape closest to how human raters work in most labelling pipelines, which makes calibration to humans (Chapter 5) straightforward.

**Pairwise (preference) scoring.** The judge sees the *same prompt with two responses* and picks a winner, or declares a tie. "Given this user question, which reply is better: A or B?" The score for a *system* is not a per-item number; it is a rating derived from many head-to-head comparisons across systems, aggregated via Bradley-Terry, ELO, or a similar model (Chapter 6). This is the shape the Chatbot Arena leaderboard uses, and the shape most useful when you want to *rank* candidate models, judges, or prompts against one another rather than measure any single one on an absolute scale.

Neither mode is universally better. They fail differently and cost differently:

- Absolute scoring gives you a per-item, per-system number that you can slice, alert on, and diff. It is more sensitive to *rubric drift* (a small wording change moves the mean; Chapter 2) and to the judge's absolute calibration (does "4/5" from this judge mean the same thing across releases?).
- Pairwise scoring is more robust to rubric ambiguity, because the judge only has to make a *relative* comparison, and small biases often cancel across many comparisons. In exchange it gives up per-item interpretability (there is no per-example "score"), requires more comparisons to distinguish close systems (O(N²) in the number of systems, at minimum), and inherits its own systematic bias — position bias (Chapter 4) — that absolute scoring does not have.

A useful default: use *absolute* scoring when the goal is a monitorable metric on a specific behavior (safety label, refusal rate, faithfulness score) and *pairwise* scoring when the goal is to rank candidate systems for a general "how good is this assistant" question. Many production stacks run both, using pairwise for release-time model bake-offs and absolute for per-slice regression tracking.

## Where the field settled on this

The pattern is not new but its current shape is. The paper that made "LLM-as-judge" a first-class concept in the LLM eval literature is Zheng et al. 2023 ("Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena"), which formalized both the absolute mode (a 1–10 rubric on multi-turn dialog, MT-Bench) and the pairwise mode (Chatbot Arena's crowdsourced head-to-heads scored by GPT-4). The same paper is where the field's baseline numbers for judge-human agreement (~80% on MT-Bench-style tasks, roughly matching human-human agreement) come from, and where the systematic biases we spend Chapter 4 on were first characterized end-to-end.

Since then the space has grown along four axes worth naming, because the rest of the module maps onto them:

1. **Rubrics with anchors** (Chapter 2 and Chapter 3). The failure mode of an unanchored rubric — "rate this from 1 to 10" with no per-score description — is that different judges, and even the same judge on different days, use the scale differently. Anchored rubrics tie each score point to a specific description ("5 = fully addresses the question with no factual errors"), which is what makes judge scores comparable across runs.

2. **Bias controls** (Chapter 4). Position bias (a judge preferring whichever response appears first), length bias (preferring longer responses regardless of quality), and self-preference (a judge scoring its own family's outputs higher) are all well-measured effects with cheap standard mitigations. Running a judge without those mitigations is publishable-quality methodological carelessness at this point.

3. **Calibration to humans** (Chapter 5). Every claim "our judge scores this system at X" needs a companion measurement of "and here is the judge-vs-human agreement on this task, so you know how much to trust X." The mechanical part is Cohen's κ, Krippendorff's α, or Spearman/Kendall correlation depending on the rubric. The methodological part is deciding what agreement level makes a judge shippable.

4. **Judge specialization and cost** (Chapter 7). The default judge for most teams is a frontier hosted model (GPT-4-class, Claude-Opus-class). A parallel line of work has produced smaller *dedicated* judges — Prometheus, JudgeLM, PandaLM — trained specifically to score rubric responses, that can reach frontier-judge-level agreement on many tasks at a fraction of the cost. The routing question (which judge to send which item to) is now a real production decision, not a research curiosity.

## The four objects, translated for judges

The four-objects taxonomy from mod-104 Chapter 1 still applies; the vocabulary just moves.

- **Model adapter.** Same as before, but you now have *two* — a *subject* adapter for the model being evaluated, and a *judge* adapter for the model doing the scoring. These are often different providers, different context lengths, and different pricing tiers.
- **Task definition.** Now includes a *rubric* — the judge's prompt template, its scoring scale, its scoring anchors, and its input variables (`{question}`, `{response}`, `{reference}`, in pairwise mode also `{response_a}`, `{response_b}`). The rubric is the load-bearing artifact; a rubric change is a task-version bump.
- **Request type.** Almost always *generation* on the judge side (rubric-in, verdict-out). Log-likelihood judging (rank continuations by probability) exists as an efficiency trick but is not the dominant shape. On the subject side it depends on the underlying task.
- **Scorer and aggregator.** A parser that extracts the verdict from the judge's generation (a letter, a number, a JSON blob), an optional bias-control aggregator (swap-and-average for pairwise, length normalisation for absolute), and a system-level aggregator (mean and CI for absolute; Bradley-Terry / ELO for pairwise).

Every judge platform — OpenAI evals' `modelgraded`, `lm-eval-harness`'s `llm_judge` output type, Inspect's `model_graded_qa` scorer, Prometheus's evaluation prompts — is a different ergonomic choice about how those four objects wire together for the judge case. Recognizing them means you can read a new judge platform's docs in an hour.

## Why this module is not "just prompt engineering"

There is a persistent framing that model-graded evaluation is a matter of writing a good rubric prompt, and that everything else is downstream. That framing is wrong in a specific and expensive way. The prompt matters, but the reason judges *fail* in production almost never traces back to the prompt alone:

- A well-worded rubric with no anchors still drifts across runs.
- A well-anchored rubric with no position-bias control still produces the wrong ranking in pairwise mode.
- A judge with well-controlled biases still ships bad numbers if it was never calibrated to humans on your specific task.
- A calibrated judge still bankrupts you if you route every item to a frontier model when a small dedicated judge would do.

The skills in this module — rubric design with explicit anchors (Chapters 2–3), swap-and-average and length-normalisation controls (Chapter 4), κ / α / Spearman-based calibration studies (Chapter 5), Bradley-Terry aggregation (Chapter 6), and judge tier routing (Chapter 7) — are the ones that separate a judge you can defend from a judge that produces a plausible number until it doesn't. Every one of them has a standard technique. None of them are optional if the number is going to gate a decision.

## Summary

An LLM judge is a measurement instrument built out of another model, and it operates in one of two modes: *absolute* scoring, which returns a per-item score against a rubric and aggregates by mean, and *pairwise* scoring, which returns a winner between two candidate responses and aggregates by a rating model. Absolute mode is easier to slice, alert on, and calibrate; pairwise mode is more robust to rubric ambiguity and better for ranking systems. Both inherit systematic biases (position, length, self-preference) with well-known mitigations, both require calibration against a human gold set before their numbers can gate decisions, and both live inside a growing ecosystem of judge platforms and dedicated judge models with real cost-versus-quality trade-offs. The next chapter opens the first of those questions: how to design an absolute rubric whose scores mean the same thing today, next week, and across judges.
