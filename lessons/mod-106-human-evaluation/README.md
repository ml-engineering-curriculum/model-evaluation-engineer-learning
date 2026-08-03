# mod-106-human-evaluation: Human Evaluation: Annotator Workflows, Agreement, and Gold Sets

**Estimated effort:** 12 hours

The previous module treated a second capable model as the scoring function. Every κ number that module reported was measured against a set of human labels that someone had to produce first, under a specification, in a tool, by a person on the other end. This module is about how those labels come into existence: the annotation guidelines annotators actually read, the pilot rounds that shake out the ambiguity in a v0 spec, the agreement statistics that tell you whether the labels are reliable, the adjudication and gold-set discipline that keeps them reliable over time, the side-by-side UX that shapes what annotators actually click, and the sourcing decision — crowd platform, in-house team, or expert panel — that determines who is on the other end at all.

The mental error to avoid is treating human labels as the "true" answer that model scoring approximates. A human annotator is also a measurement instrument, with systematic error, drift, and cost. The point of this module is to make human labels *reliable, auditable, and reproducible enough* that every downstream number they support — judge calibrations, leaderboards, regression gates, safety audits — can defend the reliability claim it is making.

## Learning objectives

- Design annotation instructions, pilot annotation, and a calibration round.
- Compute Cohen's / Fleiss' κ and Krippendorff's α, and pick the right one per labelling shape.
- Run an adjudication procedure and build a gold-set rotation that resists annotator drift.
- Design a side-by-side comparison UX with attention-check items and bot-detection.
- Make the build-vs-buy decision across crowd (Scale, Surge, Mercor, Prolific) and in-house annotator setups, with the data-residency and cost trade-offs explicit.

## Lecture chapters

1. [`01-why-humans-still-label.md`](01-why-humans-still-label.md) — where humans are still the scoring function, the four objects translated for human eval, and the gold / silver / pilot vocabulary that structures the rest of the module.
2. [`02-writing-annotation-instructions.md`](02-writing-annotation-instructions.md) — the five sections and seven writing rules that separate guidelines that produce κ ≥ 0.6 from those that produce κ ≤ 0.4, plus the training package (qualification quiz, worked examples, reference sheet, feedback channel) that ships with the guideline.
3. [`03-pilot-and-calibration-rounds.md`](03-pilot-and-calibration-rounds.md) — the pilot round that finds the ambiguity in a v0 guideline, the calibration round that verifies the revised v1 meets a pre-committed agreement threshold, and the one-page checklist for both.
4. [`04-agreement-statistics-kappa-and-alpha.md`](04-agreement-statistics-kappa-and-alpha.md) — Cohen's κ, weighted κ, Fleiss' κ, and Krippendorff's α: what each corrects for, when to use which, how to compute them in `sklearn` / `statsmodels` / `krippendorff`, why raw percent agreement is misleading, and how to read a κ number against task stakes.
5. [`05-adjudication-and-gold-set-rotation.md`](05-adjudication-and-gold-set-rotation.md) — the written adjudication procedure (who, when, what is recorded), the adjudication rate as a health signal, and the gold-set discipline — blind injection, per-annotator accuracy over rolling windows, impression-based retirement, and forced re-adjudication at guideline major-version bumps.
6. [`06-side-by-side-ux-and-attention-checks.md`](06-side-by-side-ux-and-attention-checks.md) — the four load-bearing UX decisions for a pairwise comparison (per-item randomization, ternary verdict, blinded system identity, no post-submit revision) and the attention-check + bot-detection primitives (explicit-instruction, trivial-correctness, duplicate-consistency; time-per-item, response entropy, platform reputation).
7. [`07-build-vs-buy-crowd-vs-in-house.md`](07-build-vs-buy-crowd-vs-in-house.md) — the nine axes on which the sourcing decision is made (cost per label, latency, throughput, quality, drift, specialization, data residency, lock-in, auditability), the four recurring configurations (fast crowd, managed vendor, expert panel, in-house), and the decision procedure with an honest cost model.

## Exercises

Four hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after the chapters it depends on.

- [`exercise-01-annotation-guidelines-and-pilot.md`](exercises/exercise-01-annotation-guidelines-and-pilot.md) — author a v0 annotation guideline for a real task, run a 50-item pilot with at least two annotators, produce v1 with a written changelog.
- [`exercise-02-iaa-computation-and-disagreement-analysis.md`](exercises/exercise-02-iaa-computation-and-disagreement-analysis.md) — implement Cohen's / weighted κ, Fleiss' κ, and Krippendorff's α from scratch and against library baselines, then apply them to a real multi-annotator label set with bootstrap CIs, confusion matrices, and named disagreement modes.
- [`exercise-03-adjudication-and-gold-set-rotation.md`](exercises/exercise-03-adjudication-and-gold-set-rotation.md) — design and simulate a gold-set injection + rotation system that detects annotator drift, with the operational monitoring dashboard specced out.
- [`exercise-04-side-by-side-ux-with-attention-checks.md`](exercises/exercise-04-side-by-side-ux-with-attention-checks.md) — build a minimal side-by-side pairwise annotation UI with per-item randomization, ternary verdict, three attention-check patterns, and a bot-detection heuristic, then run a small self-study to verify the QC primitives fire.

Reference solutions live in the paired `model-evaluation-engineer-solutions` repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for primary references — Cohen 1960 and 1968 (κ, weighted κ), Fleiss 1971 (multi-rater κ), Krippendorff's *Content Analysis* (α), Landis and Koch 1977 (κ interpretation bands), Feinstein and Cicchetti 1990 (κ paradox), Artstein and Poesio 2008 (a standard survey of IAA in NLP), Zheng et al. 2023 (Chatbot Arena human pairwise study), Veselovsky et al. 2023 (LLM-mediated crowd labelling), plus the vendor documentation (Prolific, Scale, Surge, Mercor) and the annotation-tool documentation (Label Studio, Prodigy, Argilla, Doccano, Inception).
