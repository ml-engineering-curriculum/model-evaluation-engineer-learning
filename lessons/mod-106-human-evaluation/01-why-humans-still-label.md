# Why Humans Still Label (And What a Human-Eval Pipeline Actually Is)

The previous module treated a second model as the scoring function. It is a legitimate move — for many tasks a calibrated judge is the cheapest measurement that clears the quality bar. But every calibration study, every rubric anchor, every claim that "the judge agrees with humans at κ = 0.72" is standing on a set of *human labels* that had to exist first. This module is about how you produce those labels: the instructions annotators read, the pilots that shake out the ambiguity, the agreement statistics that tell you whether the labels are reliable, the adjudication protocol that turns disagreements into ground truth, the UX that keeps a crowd honest, and the vendor decision that determines who is actually clicking the buttons.

The mental error to avoid is treating human labelling as the "true" answer that model scoring only approximates. A human annotator is *also* a measurement instrument, with systematic error, drift, and cost. The point of the discipline in this module is not to make human labels perfect. It is to make them *reliable, auditable, and reproducible enough* that when the judge's κ against them is 0.72, that number means something. Everything else in evaluation — leaderboards, calibration studies, regression gates, safety audits — is downstream of that guarantee.

## Where humans are the scoring function

Three families of eval questions still route to humans by default.

**Judgments that require domain expertise the model cannot reliably self-check.** Grading a legal summary against a jurisdiction's precedent. Rating a diagnostic differential against clinical practice. Scoring a piece of code for idiomaticity in a codebase's house style. LLM judges plateau on tasks where the rubric itself requires expert knowledge that the judge model does not have, and no amount of prompt engineering fixes that. This is the "expert annotator" quadrant — small panels of qualified raters, high cost per label, no crowd substitute.

**Judgments that will be used to *calibrate* a judge.** Every LLM-as-judge deployment described in mod-105 requires a human gold set to compare against. If the judge is going to score millions of items in production, the calibration set of a few hundred human-labelled items is what determines whether those millions of scores are trustworthy. The gold set is a fixed investment that amortizes across every future judge run. Cutting corners on it makes every downstream number weaker.

**Judgments where reference-free preference is what matters and pairwise is the shape.** Chatbot Arena is the canonical example — a user prompt with two model responses and a human picks which one they'd rather have received. There is no reference answer, no rubric criterion that would fully capture "better," and the aggregate rating from many humans is exactly the metric being measured. Model judges can approximate this (mod-105 Chapter 6), but the ground-truth signal against which any judge is calibrated is still the human vote.

Other tasks that used to require humans — reference-based accuracy on multiple-choice, exact-match on extractive QA, log-likelihood on standardized continuations — have migrated to automated scoring or to LLM judges. The remaining human-eval work is proportionally more valuable per label and needs proportionally more discipline. This is why a role that used to be "we hired some annotators" is now a first-class engineering surface with its own tooling stack.

## The four objects, translated for human eval

The taxonomy from mod-104 Chapter 1 still applies; the vocabulary shifts one more time.

- **Annotator adapter.** The equivalent of the model adapter is the *labelling interface* — the tool an annotator uses to record a judgment. Label Studio, Argilla, Prodigy, Doccano, Inception, a custom internal UI, a Scale AI project, a Prolific survey. What varies: what task types it supports (span, choice, ranking, free-text), what quality-control primitives it offers (gold items, attention checks, review queues), what it exports (per-annotator labels, timestamps, revision history).
- **Task definition.** A written annotation guideline (the human-facing equivalent of a rubric), a set of examples for each label, an explicit list of edge cases and how to resolve them, a versioned document that annotators are trained against. The guideline is the load-bearing artifact; a guideline change is a task-version bump and forces at least a re-check on the gold set.
- **Request type.** The shape of the judgment: categorical single-label, categorical multi-label, ordinal on a scale, span selection over a passage, pairwise preference with optional tie, ranking over N candidates, free-text critique. Each shape has a matching agreement statistic (Chapter 4).
- **Scorer and aggregator.** Per-item aggregation — how you turn multiple annotators' labels on the same item into a single "consensus" label (majority vote, adjudication, weighted vote). System-level aggregation — how you turn consensus labels into a metric. Inter-annotator agreement statistics live here; they are not the metric, they are the *diagnostic on whether the metric is trustworthy*.

Every human-eval platform is a different ergonomic choice about how these four objects wire together. Recognizing them makes a new tool's docs an afternoon of reading.

## The gold / silver / pilot vocabulary

Three overlapping terms show up in every human-eval spec and mean specific things.

**Gold set** — a fixed, high-confidence set of items whose "correct" labels are known (usually via careful multi-annotator adjudication) and that is used to measure ongoing annotator quality, detect drift, and estimate accuracy. Items appear inline in the annotator's queue but the annotator does not know which items are gold. A per-annotator accuracy on gold is the primary quality signal for crowd work. Chapter 5 is about designing and rotating this set so annotators cannot memorize it.

**Silver set** — labels produced by the pipeline itself (adjudicated multi-annotator labels or single-annotator labels on non-critical items) that are treated as ground truth for downstream use even though they were not adjudicated to gold-set standards. Bulk training data typically sits at silver. The tradeoff is scale versus reliability.

**Pilot set** — a small (50–200 item) preliminary batch used to shake out ambiguity in the guidelines *before* scaling to production. The pilot is not gold. Its purpose is to surface disagreements between annotators, feed those disagreements back into the guidelines, and produce the first version of the guideline document that survives contact with real data. Chapter 3 is the full pilot procedure.

These distinctions matter because they map to different quality controls. A gold set is designed to be *diagnostic*; a silver set is designed to be *cheap enough at volume*; a pilot set is designed to be *iterated on*. Confusing them — running a pilot as if it were gold, or treating silver labels as if they were adjudicated — produces evaluations whose reliability nobody can defend.

## The two decisions this module trains you for

Everything downstream in the module comes back to two go/no-go decisions you will run repeatedly on the job.

**"Are these labels reliable enough to use?"** — the reliability decision. The instruments are inter-annotator agreement statistics (Cohen's κ, Fleiss' κ, Krippendorff's α, weighted κ; Chapter 4), per-annotator accuracy on gold, guideline pilot outcomes, and adjudication rates. The decision is whether the label set is trustworthy enough to serve as ground truth for a calibration study, a leaderboard, or a training signal — and if not, whether the fix is in the guidelines, the annotator pool, or the task itself.

**"Where do the annotators come from?"** — the sourcing decision. The instruments are cost per label, latency to first batch, quality on qualifying tests, data-residency constraints, and vendor lock-in risk. The decision is between paid crowd platforms (Prolific, Scale, Surge, Mercor), managed in-house teams, or specialist expert panels. Chapter 7 lays out the tradeoffs.

Every subsequent chapter answers one piece of one of these questions. The exercises put them end-to-end.

## Summary

Human evaluation is where every other evaluation ultimately grounds out — the judge you calibrate against humans, the leaderboard whose ranking comes from human pairwise votes, the training set an annotator wrote the label for. A human annotator is a measurement instrument like any other, with systematic error, drift, and cost, and this module is about the discipline that makes their labels reliable enough to defend. That discipline decomposes into the four familiar objects (labelling interface, guideline, request shape, aggregator), the gold / silver / pilot vocabulary for what a label set is *for*, and two persistent go/no-go decisions: reliability (are these labels trustworthy?) and sourcing (who is producing them?). The next chapter opens the first of the two by writing an annotation guideline that survives a pilot round.
