# Adjudication and Gold-Set Rotation

Chapter 4 gave you a number: κ = 0.66, say, on a calibration set. That number is a *diagnostic* about the label-production process. It is not the labels you ship. To get a shippable label set you have to resolve the disagreements the agreement statistic surfaces — item by item — into a single "consensus" label per item. That is adjudication. And once you have those adjudicated labels, you have to keep them working *over time*, because annotators drift, guidelines evolve, and a set that was gold six months ago has degraded by unknown amounts. That is the gold-set rotation problem. This chapter is the operational discipline around both.

## Adjudication as a written procedure

Adjudication is the step where an authoritative labelling decision is made for items on which annotators disagreed. Every annotation pipeline needs one, and the one you don't write down explicitly is the one that quietly turns into "whoever felt strongest in the moment wins" and ships biased labels.

A working adjudication procedure has three parts.

**Who adjudicates.** One of:

1. **A third annotator** — draw from the same qualified pool, blind to the two existing labels, produce a label with the same guidelines. If the third label matches one of the original two, that label wins (2-of-3). If the third label matches *neither* original label, the item goes to a fourth mechanism.
2. **A domain expert / project lead** — a senior annotator or the task designer reviews the item, both annotator labels, both annotator notes, and produces a label with a written rationale. Slower per item than the third-annotator route; higher confidence on the outcome; the natural default for high-stakes labels.
3. **Discussion between the two original annotators** — the two annotators meet (synchronously or async in the tool), talk through their reasoning, and arrive at a consensus label. Cheapest; least defensible on its own for high-stakes labels because social dynamics distort outcomes; useful primarily when the "correct" label is genuinely a judgment call and the disagreement is over interpretation of the guideline rather than the facts of the item.

Real projects use combinations. A common structure: third annotator by default; escalate to expert on items where either (a) the third annotator also disagrees, (b) any annotator flagged the item as ambiguous, or (c) the item is drawn from a high-stakes slice.

**When to adjudicate.** The trigger conditions to write down:

- Any item where two initial annotators produced different labels.
- Any item where either annotator's confidence (if you collect one) is below a threshold.
- Any item flagged with an "escalate" note by an annotator.
- On categorical tasks with more than three labels, any item where the two disagreeing labels are non-adjacent (a 1 vs. 4 disagreement almost always requires expert adjudication, not just a third rater).

Items where both annotators produced the same label do not go through adjudication — they are the shippable consensus by construction, and the fraction of items that reach this state is a useful summary statistic ("77% of items had annotator consensus; 23% required adjudication").

**What the adjudicated record looks like.** The output row per item is at minimum:

```
id, annotator_a_label, annotator_b_label, adjudicated_label,
adjudicator_id, adjudication_method, adjudication_reason
```

The `adjudication_reason` field is the one people skip. Write it. One sentence. "Annotator A read `refused` as `not_helpful`; the guideline reserves `not_helpful` for direct wrong answers, so the correct label under v1.2 is `helpful` with a `refused: safety` note." Future you, and the next person owning this guideline, will use those reasons as the source material for the next round of guideline revision.

## Adjudication rate as a health signal

The fraction of items that require adjudication is itself a useful measurement, not just a workload metric. It tells you where the guideline is not producing consensus, and where in production you should expect additional review cost.

Track adjudication rate:

- **Overall** — a rate rising over time on a stable guideline usually indicates annotator drift or new item distribution.
- **Per slice** — a high rate on one topic or one response format usually indicates a guideline gap for that slice.
- **Per annotator pair** — a high rate isolated to one pair suggests one annotator is drifting relative to the pool.

A stable adjudication rate of 15–30% is normal on most non-trivial tasks. A rate above 40% suggests the guideline needs work (Chapter 3's pilot revision); a rate below 5% either indicates a very well-specified task or annotators copying each other (which happens on some tools where labels are visible cross-annotator by default — a bad configuration; disable it).

## The gold set: what it is and what it is for

A **gold set** is a set of items with high-confidence "correct" labels — typically produced by careful adjudication with expert review — used to measure ongoing annotator quality and to detect drift. Its purpose is *not* to be a training set, an evaluation set, or a benchmark; it is a *quality-control instrument*.

Two properties are load-bearing.

**Blind injection.** Gold items are injected into the annotator's queue *without the annotator knowing they are gold*. From the annotator's perspective, a gold item looks exactly like a normal work item. This is the only way per-annotator accuracy on gold measures actual annotation quality rather than "how well the annotator does on items they know are being watched."

**Per-annotator accuracy tracked over time.** The primary metric is per-annotator accuracy on gold, computed on a rolling basis (last 50 gold items an annotator saw, say, or last 30 days). A drop in this metric is the earliest signal that:

- A specific annotator is drifting or has stopped following the guideline.
- The guideline is out of sync with the current item distribution (multiple annotators drop together).
- Gold items have been leaked or memorized (see rotation below).

Most crowd platforms and dedicated annotation tools have built-in gold-item support. Label Studio, Prodigy, Scale, Surge all support injecting known-answer items and computing per-annotator accuracy over them. If you are building your own tool, treat gold-item support as day-one, not a v2.

## Sizing and rotating the gold set

Two questions determine how the gold set holds up over time: how big it should be and how often you should rotate it.

**Size.** Enough gold items that a per-annotator accuracy on the last N golds has a usable confidence interval:

- 30–50 gold items minimum per rolling window per annotator. Fewer than this and per-annotator accuracy is too noisy to alert on.
- Injection rate typically 5–10% of the annotator's queue. Higher rates burn gold quickly (see rotation); lower rates delay drift detection.
- Total pool size 5–10× the per-annotator window, so that gold items in circulation are not always the same 30 for every annotator.

**Rotation.** Gold items degrade in two ways:

1. **Memorization.** An annotator sees the same gold item three times and remembers what they labelled it, or infers from item characteristics ("all short items about widgets are `not_helpful`") what the correct answer is. Their accuracy on that item then measures memory, not annotation.
2. **Guideline drift.** As the guideline evolves through pilots and edge-case additions, the "correct" label on an old gold item may change or become ambiguous under the new guideline. A gold item labelled under v1.0 may not be a valid gold under v1.4.

The rotation policy has three components:

1. **Retire gold items after N impressions per annotator.** N = 2 or 3 is common. Track impressions per (annotator, gold_item) pair.
2. **Rotate a fixed fraction of the pool per period.** For example, 10% of gold items per month are replaced with newly-adjudicated items. This keeps the pool fresh even if individual retirement is slow.
3. **Force a full re-adjudication of the pool at each guideline major version bump.** When the guideline goes from v1.x to v2, every gold item is re-adjudicated by expert against the new guideline; some pass, some are updated, some are retired. Do not carry old golds silently across major guideline versions.

The metric that tells you the rotation policy is working: **per-annotator gold accuracy stays statistically flat over time on a stable guideline**. If it slowly rises over months, annotators are memorizing and the rotation is too slow. If it slowly drops, the guideline is drifting away from the golds and you need re-adjudication.

## Drift detection: the operational shape

Concretely, the drift-detection loop looks like:

```
Nightly (or weekly, depending on volume):
  For each active annotator A:
    - Compute A's accuracy on the last 50 gold items they saw
    - Compare to A's baseline (rolling 90-day average)
    - If accuracy drops > 10 percentage points, flag A for review

  Compute pool-wide gold accuracy (mean across all active annotators)
    - If pool accuracy drops > 5 percentage points over a 2-week
      window, flag the guideline for review — the drop is
      distributed rather than isolated to one annotator

  For each gold item:
    - If aggregate accuracy across annotators on this item drops
      below 70% for two consecutive weeks, flag the item for
      re-adjudication — the correct label under the current
      guideline may have changed
```

The three flags map to three different remediations: an individual annotator flag triggers a retraining session or removal from the pool; a pool-wide flag triggers a guideline review; an item-level flag triggers re-adjudication of that gold item. Conflating the three is the common mistake — retraining an annotator when the guideline is what changed does not fix the problem.

## Adjudication and the gold set as a feedback loop

The relationship between adjudication and the gold set is not one-directional. Adjudicated items in the production pipeline are the candidate pool from which gold items are drawn (specifically: items that were adjudicated with expert review, where the correct label under the current guideline is unambiguous). Gold items whose aggregate accuracy drops feed back into the guideline revision process.

A running annotation project therefore has three intertwined loops:

1. **Item flow:** production items → 2+ annotators → agreement check → adjudication if needed → adjudicated label → downstream use.
2. **Gold flow:** adjudicated items with high confidence → gold set → blind injection into annotator queues → per-annotator accuracy → drift alerts.
3. **Guideline flow:** disagreements, escalations, gold accuracy drops → guideline revision → new version → re-adjudication of stale golds → updated gold set.

The whole system is what a reviewer means when they ask "how do you know your labels are still reliable six months from now?" Any single loop without the others is fragile. All three together are what makes the reliability claim defensible past the initial calibration study.

## Common failure modes

**Adjudication done by whoever is around, no procedure.** The resulting adjudicated labels vary in quality by adjudicator; downstream κ against the adjudicated set is inflated because the adjudicator is essentially a third source of noise treated as ground truth. Fix: written procedure, small adjudicator pool, adjudicator identity logged.

**Gold set never rotated.** Per-annotator accuracy on gold quietly rises over months as annotators memorize; the accuracy metric no longer detects drift because it is measuring memory. Fix: retirement policy, per-annotator impression tracking, forced rotation.

**Gold items visible in the interface as "gold."** Some tools default to marking gold items in the UI for adjudicator convenience. Turn this off for the annotators; leave it on only for the reviewer view. Otherwise annotators will label gold items with more care than production items and the accuracy metric is meaningless.

**Guideline version and gold set version not linked.** Gold items get created against a v1 guideline, the guideline goes to v2, the gold set is not re-adjudicated, and now the "correct" labels on some golds are wrong under the current spec. Fix: guideline major-version bumps trigger full gold re-adjudication; log the guideline version each gold item was adjudicated under.

**Adjudication rate ignored.** The project runs for months with a 45% adjudication rate that nobody looks at, when a 45% rate is a signal that the guideline is fighting the annotators. Fix: adjudication rate on the same dashboard as κ and per-annotator gold accuracy.

## Summary

Adjudication is the step where inter-annotator disagreements become shippable labels; the procedure — who adjudicates (third annotator, expert, discussion), when (disagreement, low confidence, escalation flag, wide-gap disagreement), and what is recorded (the reason field is not optional) — is written down before labelling begins, not improvised per item. The adjudication *rate* itself is a health signal: overall, per-slice, and per-annotator-pair. The gold set is the ongoing quality-control instrument that lets you detect annotator drift and guideline drift once production is running; it works only if items are blind-injected, rotated on an impression-based retirement policy, and re-adjudicated at each guideline major-version bump. The whole thing — adjudication, gold set, guideline revision — is a three-loop system, and the reliability of a label set six months into production is the property of the system, not of any initial calibration number. The next chapter shifts from what happens after annotators disagree to how the interface they use shapes their labels in the first place — specifically, the pairwise side-by-side UX and its attention-check and bot-detection primitives.
