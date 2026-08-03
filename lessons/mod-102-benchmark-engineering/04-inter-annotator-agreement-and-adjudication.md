# Inter-Annotator Agreement, Adjudication, and Gold Rotation

Chapter 3 ends with a completed pilot: annotators have labelled a small round using the revised instructions, and you have a per-item table of labels. This chapter is what you do with that table — how to measure agreement in a way that separates signal from chance, how to adjudicate the residual disagreements into a defensible gold label, and how to keep the gold set healthy after it has been sitting in production for months.

Cohen (1960), Fleiss (1971), and Krippendorff (2004) developed the agreement statistics; adjudication and rotation are the operational wrapper around them.

## Why raw agreement is not enough

The first-instinct metric — "annotators agreed on 90% of items" — collapses two very different situations. If the label is binary and the true prevalence is 90% positive, two annotators who both always answer "positive" agree 100% of the time and have measured nothing. Raw agreement is inflated by the base rate, so a comparison across evals with different base rates is not meaningful.

Chance-corrected agreement statistics fix this. They ask: given the observed marginal frequency of each label, how much of the observed agreement is more than what two annotators labelling randomly (but with the same marginals) would produce?

## Cohen's kappa: two annotators, categorical labels

For two annotators labelling `n` items with a fixed set of `k` categorical labels, Cohen's kappa is:

    κ = (p_o − p_e) / (1 − p_e)

where `p_o` is the observed agreement (the proportion of items where the two annotators picked the same label) and `p_e` is the expected agreement under chance, computed from the annotators' marginal label distributions. Concretely, if `p_i^A` is annotator A's frequency of label `i` and `p_i^B` is annotator B's, then

    p_e = Σ_i p_i^A · p_i^B

κ ranges from -1 (perfect disagreement) to +1 (perfect agreement) with 0 as chance. The often-cited Landis & Koch (1977) benchmarks (`< 0.20` slight, `0.21–0.40` fair, `0.41–0.60` moderate, `0.61–0.80` substantial, `0.81–1.00` almost perfect) are conventional labels, not thresholds you can defend as universal. In eval work, the operational target is task- and stakes-dependent:

- **Decision-rule tasks** with a well-specified rubric: aim for κ ≥ 0.85. Anything lower usually means an unhandled edge case in the rules, not a genuinely hard construct.
- **Judgment tasks** (helpfulness, harmfulness, quality): κ = 0.70 is a strong result and κ = 0.60 is often the realistic ceiling. Report the number; don't paper over it.
- **Safety-critical evals** where a wrong label has a real cost: raise the bar to κ ≥ 0.90 and treat items below the bar as adjudication-required by policy.

Two important cautions about κ.

**Kappa is not invariant to prevalence.** On highly imbalanced label distributions, κ can be low even when agreement is high — the "kappa paradox" (Feinstein & Cicchetti 1990; Byrt et al. 1993). Report both raw agreement and κ so a reader can tell which regime they are in.

**Kappa is sensitive to the label set.** Merging two rarely used labels into one changes κ. If you compare κ across benchmarks, verify they use the same label set at the same granularity.

## Fleiss' kappa and Krippendorff's alpha: more than two annotators

When each item has three or more annotators (and the annotator identities may differ across items), Cohen's κ does not directly apply.

- **Fleiss' κ (1971).** For a fixed number of annotators per item, all interchangeable. Formula computes chance agreement from the pooled marginal frequencies. Ranges the same way as Cohen's.
- **Krippendorff's α (2004).** A more general family that accommodates missing annotations, varying numbers of annotators per item, and multiple levels of measurement (nominal, ordinal, interval, ratio). Preferred when the annotation design is uneven — which is most real projects.

Reference implementations you can trust:

- `sklearn.metrics.cohen_kappa_score(y1, y2, weights=None)` for Cohen's κ, with optional linear or quadratic weights for ordinal labels.
- `statsmodels.stats.inter_rater.fleiss_kappa(table)` for Fleiss' κ.
- `krippendorff` (Python package by `pln-fing-udelar`, MIT-licensed) or the `nltk.metrics.agreement.AnnotationTask` class for Krippendorff's α.

For an eval where every item has exactly three annotators (a common pilot design), report Fleiss' κ as the headline and Krippendorff's α with a note that they agree closely — they do not always, and a divergence is a signal that the annotation design has something uneven in it.

## Weighted kappa for ordinal labels

For Likert-style labels (1–5 quality scores, Low/Med/High severity), a disagreement between "1" and "5" is worse than between "3" and "4". Cohen's κ treats them the same. Weighted κ (Cohen 1968) applies a penalty matrix so ordinal disagreements are scored proportionally to their distance:

- **Linear weights.** Penalty proportional to the ordinal gap.
- **Quadratic weights.** Penalty proportional to the square of the gap. Standard for Likert-like scales.

`cohen_kappa_score(y1, y2, weights='quadratic')` is the one-liner. When you present a Likert κ, always say which weighting; the numbers differ substantially.

## What a low kappa is telling you

A κ below the target is diagnostic, not just a red mark. The residual — the items annotators disagreed on — is the signal. For each disagreement, categorize:

- **Instructions ambiguous.** The item is a legitimate edge case not covered by the rules. Fix the rules and re-annotate.
- **Item ambiguous.** The item itself is ambiguous — a genuinely borderline case that competent readers can differ on. Either write a rule for it or drop it from the eval; do not ship a gold label for it if the ambiguity is intrinsic.
- **Annotator error.** One annotator misapplied the rules. Feedback to the annotator; if it recurs, they may need re-calibration or removal.
- **Adversarial or degenerate item.** The item can be scored either way with equal justification (the classic "is a hot dog a sandwich" case). Drop or reclassify.

You do not just adjudicate the disagreements. You *learn from them*. The adjudication log is the primary source of edge-case additions to the instructions for the next round.

## Adjudication: producing the gold label

Adjudication is the process that takes items with annotator disagreement and produces a single, defensible gold label. Common patterns, in decreasing rigor:

- **Third-annotator + adjudicator.** Every item gets N=3 annotators. Items with unanimous agreement are gold. Items with disagreement go to an adjudicator (a senior annotator or the instruction author) who resolves them with a written rationale. The rationale is the audit trail.
- **Majority vote.** With N=3+, majority is the gold. Cheap, but drops the rationale and hides ambiguity. Use only when the adjudication cost is prohibitive; report the fraction of items that were resolved by majority vs. unanimous.
- **Consensus meeting.** The annotators meet and reach agreement on disputed items. Requires everyone to be present; produces the best rationale but does not scale to hundreds of items.
- **Expert override.** A single domain expert produces the gold; annotators are graded against the expert. Fastest but not agreement-based; you have replaced the eval's construct with "what this expert thinks."

For most eval work, third-annotator + adjudicator is the shape to reach for. Two additional rules:

**Track adjudication outcomes as data.** Each adjudicated item records: which annotators disagreed, which labels they chose, which label the adjudicator selected, and the adjudicator's written rationale. This is the input to Chapter 3's next revision of the instructions.

**Guard against adjudicator drift.** Rotate adjudicators periodically and spot-check their decisions against a peer's. An adjudicator who is themselves miscalibrated silently degrades the gold set.

## The adjudication log schema

Store adjudication decisions in a structured file, versioned with the dataset:

```yaml
- item_id: item-a4b1c
  annotator_labels:
    ann-01: helpful
    ann-02: partially-helpful
    ann-03: helpful
  adjudicator: senior-01
  gold_label: helpful
  rationale: >
    Response answers the primary question; the follow-up gap noted by ann-02
    is stylistic, not substantive. Instruction rule H3 applies.
  adjudicated_at: 2026-08-03T14:22:00Z
  instructions_version: 1.2.0
```

The `rationale` field is not optional. An unrationed adjudication is indistinguishable from an arbitrary call, and it teaches nothing for the next revision. `instructions_version` lets you filter the log to "decisions made under the current rules."

## Gold rotation: keeping the gold healthy over time

A gold set is not a permanent object. Three things move it:

- **Instruction drift.** New edge cases surface, the rules evolve, and old gold labels stop matching the current rubric.
- **Annotator drift.** The pool of annotators changes over time; the new pool's calibration is not automatically the old pool's.
- **Construct drift.** The world changes. "Helpful" in a customer-support context evolves with the product. Safety categories expand. Language use shifts.

Gold rotation is the ongoing practice of re-annotating a sample of the gold set to detect and correct these drifts. Suggested shape:

- **Rotation cadence.** Quarterly or per-major-release, whichever is more frequent.
- **Rotation sample.** A stratified sample of ~10% of the gold, plus 100% of items whose adjudication rationale invoked instructions that have since been rewritten. Weight toward slices where the model's score has moved recently, since those are the slices where a gold-label drift would show up.
- **Rotation procedure.** Re-annotate the sample with the *current* annotator pool and *current* instructions, blind to the old gold. Compute the κ between the old and new gold. Items where they differ trigger a review: is the old gold wrong under the new rules (update gold; call it a Chapter 6 minor bump), or is the new annotation wrong (annotator feedback or instruction fix)?
- **Reporting.** Publish a rotation report per cycle: `n` re-annotated, κ against previous gold, number of labels changed, categorization of the changes.

A gold set that has never been rotated on a benchmark that has been live for more than a year is a construct-validity risk. Rotation is the moving-target defense.

## Reporting IAA in the model card

The benchmark's model card / datasheet should include, at a minimum:

- Number of annotators per item.
- Annotator pool description (domain expertise, language, geography — subject to privacy).
- Instructions version at the time of annotation.
- Cohen's or Fleiss' κ (specify which), computed on the full set, with 95% CI (bootstrap over items).
- Fraction of items resolved by unanimous agreement vs. adjudication.
- Adjudicator identity or role, and the fraction of items each adjudicated.
- Rotation history: dates of rotations, sample sizes, κ against prior gold.

These numbers are how a downstream consumer decides whether the gold you shipped can carry the decisions they want to make on top of it.

## What IAA does not measure

Kappa is silent on:

- **Whether the label is correct.** Three annotators can agree on the wrong label because the instructions encoded a construct-validity error. IAA is necessary, not sufficient.
- **Whether the gold set represents the deployment surface.** Chapter 7 covers holdouts and the private test set; the gold set can be internally consistent and yet drawn from a narrow slice of production.
- **Whether the item itself was good.** Bad items with clear rulings get high agreement. Item quality is Chapters 1–2's job.
- **Long-horizon rot.** Even a perfectly-calibrated gold set drifts. Rotation is the answer, not a bigger IAA number.

## Summary

Raw agreement is inflated by base rates; use chance-corrected statistics — Cohen's κ for two annotators, Fleiss' κ or Krippendorff's α for three-plus, weighted κ for ordinal labels. Interpret κ against a task-specific target (≥ 0.85 for decision rules, ≥ 0.70 for judgment, higher for safety-critical). Below target, treat the disagreements as diagnostic and rewrite the instructions. Adjudicate residual disagreements with a written rationale and a versioned log, using a third-annotator + adjudicator pattern where you can afford it. Rotate the gold set on a cadence to catch instruction, annotator, and construct drift. Chapter 5 turns to the other side of the eval's construct-validity story: whether the items themselves have already leaked into the model.
