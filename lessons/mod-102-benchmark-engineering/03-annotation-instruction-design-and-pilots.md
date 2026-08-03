# Annotation Instructions and the Pilot Round

The `reference` column in Chapter 2's schema is where the benchmark's construct claim lives. If two competent annotators reading the same instructions produce different labels for the same item, the disagreement is a joint measurement of the annotators *and* the instructions — and Chapter 4's kappa numbers will reveal it. This chapter is about writing the instructions so that they measure the item, not the annotator, and about the pilot that tells you whether the instructions are ready before you spend the labelling budget.

Chapter 4 covers the statistics of inter-annotator agreement and the mechanics of adjudication and gold rotation. Chapter 3 is upstream: the design work you do *before* you compute a single kappa.

## The two failure modes of eval instructions

Almost every problem you see in an annotation project reduces to one of two shapes:

1. **The instructions do not decide the case in front of the annotator.** The annotator faces an ambiguous item, defaults to their own judgment, and different annotators default differently. Agreement is low because the instructions have not made the label a function of the item.
2. **The instructions decide the case, but not the way the construct requires.** Annotators agree at high rates, but the label they agree on is not the construct you claimed to measure. You get high kappa on a wrong operationalization.

The first failure shows up in Chapter 4 as low kappa. The second shows up as construct-validity trouble later (mod-101 Chapter 2). Both are prevented in Chapter 3, in the writing phase, or not at all.

## Anchor the instructions to a construct definition

Start with a one-paragraph construct definition: what is the phenomenon the label is trying to capture, in the language of the domain the eval is for? For a "toxicity" label, is it toxicity *as understood by content moderators in this product surface*, or toxicity *as scored by a general-purpose academic dataset*? Those are different constructs and produce different rubrics. Cronbach & Meehl's construct-validity language (mod-101 Chapter 2) is the vocabulary; the instructions are the operational bridge.

The construct definition is not the rubric. It is the sentence you keep re-reading when the rubric collides with an edge case: whichever choice matches the construct definition wins.

## Structure of the instruction document

A well-written eval instruction document has, in order:

- **Purpose.** One paragraph: the construct definition, the eval's downstream use, the annotator's role.
- **Label set.** The exhaustive set of possible labels, with a one-sentence gloss per label. If labels have levels (e.g. Likert 1–5, or severity Low/Med/High), define each level.
- **Decision rules.** The concrete algorithm the annotator follows. "If the item contains X, label Y; otherwise, if it contains Z, label W; otherwise, label default." Where possible, express as a decision tree or checklist rather than prose, because prose invites reinterpretation.
- **Examples per label.** Two to five worked positive examples per label plus at least one negative example. Include the reasoning, not just the label.
- **Edge cases with rulings.** A section that names the edge cases you have already encountered and the ruling you have decided on. This is the primary defense against annotator drift over time.
- **What to do when unsure.** Explicit escalation: `abstain` vs. `flag-for-adjudication`. Never leave the annotator to invent this on the fly.
- **Non-goals.** A short list of things the annotator should *not* consider. "Ignore length"; "ignore stylistic quality"; "do not use search"; "do not consult other annotators". Non-goals prevent construct-irrelevant variance from leaking in.
- **Metadata to record.** Any per-annotation metadata beyond the label: confidence, time-taken, flag-for-review.

Length target: 5–15 pages for a new eval. If your instructions are two pages, they are probably underspecified; if they are forty, they are probably a signal that the construct itself is being invented in the document and needs to be pushed back to the definition stage.

## Decision rules over free judgment

The single most impactful choice in instruction design is whether the annotator makes a judgment or executes a decision rule. Both have a place; the trade-off is agreement vs. construct richness.

- **Decision-rule labelling.** The annotator applies an explicit algorithm ("if the answer contains any of these substrings, label CORRECT"). Yields high agreement. Risks encoding the algorithm's blind spots into the benchmark forever.
- **Judgment labelling.** The annotator uses reasoned judgment guided by the construct definition. Yields lower agreement, sometimes catastrophically low without careful rubric work. Better matches complex constructs (helpfulness, harmfulness, factuality of a nuanced claim).

The engineering answer for most eval work: start with judgment for exploration, then extract decision rules from adjudicated disagreements until the residual is small. The gold set (Chapter 4) contains items where the rule and the judgment both matter, and it is the calibration surface for both.

## Writing labels that measure what you claim

A concrete example. Suppose you are building a "helpfulness" label for a customer-support LLM.

**Weak label set.** `helpful` / `unhelpful` / `neutral`. This produces low agreement because "helpful" is under-specified; annotators split on tone vs. content, on length, on politeness.

**Stronger label set with decision structure.**
- `resolved`: the response, taken alone, would let the user solve their problem without further back-and-forth.
- `partially-resolved`: the response gets the user closer but requires the user to ask a follow-up.
- `not-resolved`: the user would still be stuck.
- `off-topic`: the response addresses a different question than the one asked.
- `refuses-legitimate`: the response declines to help with a request the policy permits.
- `refuses-illegitimate`: the response declines to help with a request the policy forbids.

The stronger set trades one axis (helpful vs. not) for several — but each of the several is easier to decide, and the aggregate label ("useful outcome") becomes a computed function of the sub-labels rather than a judgment call. The kappa on the sub-labels is what you measure and improve.

## The examples-per-label discipline

Every label in your set gets:

- **Two typical positive examples.** The clear cases. If you cannot easily find two, the label is either rare (say so — small classes drive up variance and want their own reporting) or ill-defined.
- **One boundary example.** An item that is at the edge of the label. Explain the reasoning that keeps it on this side of the line.
- **One near-miss negative.** An item that a naive reader might label as this class but should not. Explain why.

The boundary + near-miss pairs are where most of your instruction-design value is. Almost every low-kappa dispute in the pilot will be over one of them.

## The pilot round

The pilot is the small labelling round you run before you commit to the full annotation budget. Purpose: catch the two failure modes above at their cheapest.

Suggested shape:

- **Size.** 50–100 items, sampled to over-represent boundary cases if you can identify them, otherwise a stratified sample of the source distribution.
- **Annotators.** At least three per item, drawn from the same pool as the main annotation. Not the instruction author. Not a colleague who watched the instructions being written. The pilot measures what a fresh annotator produces from the written instructions.
- **Instrumentation.** Time per item, per-item confidence, and a free-text "unclear because…" field. Prose complaints from the pilot become the next revision's edge-case section.
- **Output.**
  - A per-item table of `{annotator_id → label}`, hashed and stored.
  - A pilot IAA report (Cohen's κ for two-annotator pairs, Fleiss' κ or Krippendorff's α for three-plus; Chapter 4 covers the math).
  - A disagreement log: for every item where annotators disagreed, the labels chosen and the annotators' free-text comments.
  - A revised instruction document, with a changelog naming which pilot cases motivated each change.

The pilot is not a labelling round. Its output is a *better instruction document*, not a labelled dataset. Discard the pilot labels (or keep them only as a training set for a re-annotation round); do not include them in the gold set. Pilot labels were produced under instructions you know now to be under-specified.

## When to run more than one pilot

One pilot is a minimum; two is often warranted; three suggests the construct itself is unsettled.

- **Run a second pilot** if the first produced IAA below your target (typical target: κ ≥ 0.70 for judgment tasks, κ ≥ 0.85 for decision-rule tasks) *and* the disagreement log identifies specific instructional gaps that a revision can plausibly close. The second pilot uses the revised instructions and a fresh 50-item sample.
- **Stop and rethink the construct** if the second pilot's IAA is still below target and the residual disagreements are about the *phenomenon*, not the *rule* — e.g. two competent annotators genuinely differ on whether a borderline case is "harm" or not, because their notions of harm differ. This is a construct-definition problem, not an instructions problem. Fix it upstream, or split the label into multiple sub-constructs.

## Annotator recruitment and calibration

You cannot separate instruction design from annotator selection. Two considerations:

- **Domain expertise.** For a medical eval, non-medical annotators will produce high-agreement wrong answers on clinical questions. Match annotator expertise to construct.
- **Calibration training.** Before an annotator scores any item that goes into the gold set, they annotate a small calibration set (10–30 items with known gold labels), get feedback, and reach a threshold agreement. Annotators who cannot reach the threshold do not proceed. The calibration set is not the gold set — items from the calibration set never appear in the reporting eval.
- **Rotate periodically.** Annotator drift over hundreds of items is real; a mid-project calibration round catches it before it silently degrades the gold. Chapter 4 covers gold rotation as a running-quality control.

## Common bad patterns

- **Instructions written by the model developer.** The developer knows what they want the model to do, and the instructions bleed that in. Have someone else write the first draft.
- **"Use your best judgment" without any judgment scaffolding.** Judgment tasks need rubric anchors, especially at the endpoints and around the boundary.
- **Labels that overlap.** If two labels can legitimately apply to the same item, either you meant multi-label (say so) or your label set has a design bug.
- **Instructions changed silently mid-project.** Every edit is a version bump; the gold set records which instructions version produced each label. Otherwise you cannot tell why kappa moved.
- **No non-goals section.** Without one, annotators bring their own priors — the annotator who cares about grammar will down-weight ungrammatical helpful answers; the annotator who cares about brevity will up-weight terse ones.

## What the pilot buys you, and what it doesn't

The pilot catches the instruction-design failures. It does not catch:

- **Rare-class problems.** A label that appears in 1% of items will not have three examples in a 50-item pilot. Rare-class calibration needs targeted sampling; see the stretch material in Chapter 4.
- **Long-horizon drift.** Six months into a labelling project, the annotators are not the same team you piloted with. Gold rotation (Chapter 4) is the ongoing defense.
- **Construct drift after release.** The construct itself moves as the world moves — new categories of prompts, new harms, new formats. That is a benchmark deprecation question, not an instruction-design one; Chapter 6 covers it.

## Summary

Well-designed annotation instructions are the difference between measuring an item and measuring an annotator. Anchor them to a construct definition, structure them as a purpose paragraph plus a label set plus decision rules plus worked examples plus edge-case rulings plus non-goals. Run a pilot of 50–100 items with three or more fresh annotators; use the disagreement log to rewrite the instructions, not to label the gold set. Stop and rethink the construct — not the instructions — when a second pilot still misses the IAA target on the phenomenon itself. Chapter 4 picks up from a completed pilot and shows how to compute agreement, adjudicate the residual, and keep the gold set healthy over time.
