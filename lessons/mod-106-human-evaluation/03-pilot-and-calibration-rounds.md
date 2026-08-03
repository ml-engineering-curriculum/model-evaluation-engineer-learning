# Pilot Annotation and the Calibration Round

A v0 guideline is a hypothesis about what the annotators will do. A pilot is the experiment that tests it. The failure mode this chapter is designed to prevent is the one where you skip the pilot, run the guideline against a full batch, and discover in the aggregate agreement statistics that annotators were interpreting the task three different ways — with a bill for several thousand labels of which some meaningful fraction now need to be relabelled. Pilots are cheap; retroactive relabelling is not. The whole discipline is a small structured investment up front that saves large unstructured cleanup later.

There are two rounds worth naming distinctly. The **pilot round** exists to find and fix the ambiguity in the guidelines. The **calibration round** exists to establish that the annotators, on a version of the guideline that survived the pilot, agree well enough to enter production. Some teams collapse the two into one. That works when the guideline is a minor revision of one already deployed. It does not work on a new task.

## The pilot round: purpose and design

A pilot is a small (50–200 item) batch labelled by *all* the annotators who will be in the production pool, on a guideline that has never been used in production before. Its output is not the labels — those are typically not used. Its output is:

1. A revised guideline, `guidelines_v1.md`.
2. A written record of every disagreement and how the guideline was updated in response.
3. An initial estimate of inter-annotator agreement (a κ number with a wide CI, because the sample is small — useful as a directional signal, not as the ship criterion).
4. An initial estimate of per-item labelling time, which drives cost projections.

The pilot's design is dominated by two choices: the sample and the annotator pool.

**The sample.** Draw the pilot items to *maximize your chance of finding edge cases*, not to be representative. Include:

- Items drawn from the extremes of the natural distribution (very short and very long inputs, the tails of any easily-computed feature).
- Items you *suspect* will be hard — anything the guideline authors were unsure about while writing.
- A handful of "obviously easy" items as sanity anchors — if annotators disagree on these, the guideline is deeply broken.
- Items from any subgroup you know exists in production (different topics, different response formats, different languages).

Pilot samples that are drawn uniformly at random miss edge cases in inverse proportion to how rare they are, which is exactly the wrong direction. You are trying to *find* the failure modes.

**The annotator pool.** Every annotator who will label in production should label the pilot. For a small in-house team of 3–5, that is trivial. For a crowd platform, you'll typically qualify a larger pool through the quiz (Chapter 2) and pilot with 5–10 of them. The pilot is *not* the place to stress-test how a crowd of 200 will behave; that is the calibration round.

Every pilot item is labelled by every pilot annotator. This produces a full N-annotator agreement matrix per item, which lets you compute per-item agreement and rank items by how contested they are — the ranked list is your queue for the guideline-revision session.

## The pilot-revision session

The productive shape of the post-pilot review is a synchronous session with the guideline authors and — if possible — one or two of the pilot annotators. It runs one item at a time down the ranked-disagreement list:

1. **Read the item.**
2. **See what each annotator picked.**
3. **Ask the annotators (or reconstruct from notes) *why*.**
4. **Decide: which label is correct under the *current* guideline?** If the guideline is unambiguous but the annotator misapplied it, that is a training / feedback issue, not a guideline issue.
5. **Decide: does the guideline as written *actually* support the correct label unambiguously?** If not, revise the guideline. Add an edge-case rule, tighten a definition, add a worked example.
6. **Write down the revision in the changelog.**

You will typically get through the top 20–30 disagreeing items in a two-hour session. That is enough to move the guideline from v0 to v1 with a meaningful revision list. Do not try to resolve *every* disagreement in a single pass; the second-tier disagreements often disappear once the top-tier revisions land.

**Non-goals of the session.** Do not decide that "the annotators need to try harder." Do not decide that "the task is intrinsically hard, so κ = 0.4 is fine." Both are common escape hatches; both are ways to skip the work. The premise of the pilot is that most disagreement is fixable in the guideline; if a specific disagreement genuinely is not (rare, but real), name it, quantify it, and note that this residual disagreement is a *feature of the task* rather than a specification defect. The calibration round is what confirms that claim.

## The calibration round: purpose and design

After the pilot, the guideline is v1. The calibration round is where you verify that annotators, on v1, achieve agreement above the threshold you need — *before* you commit to a large production batch. This is the go / no-go gate that decides whether you spend $10k on the full labelling or $500 on another pilot revision.

The design differs from the pilot in three ways:

**Sample.** The calibration set should be *representative of the production distribution*, not oversampled for edge cases. This is because the κ you report from it is the number you will defend to reviewers as "the agreement on this task"; that number needs to reflect actual future work, not a stress test. 100–300 items, drawn as uniformly as your production distribution supports.

**Overlap structure.** Not every item needs every annotator. A common structure is *partial overlap*: every item is labelled by 2–3 annotators, with the specific 2–3 varying across items so that every pair of annotators shares some overlap. This lets you compute pairwise κ per annotator pair (a diagnostic — is one annotator consistently disagreeing with the rest?) and Fleiss' κ or Krippendorff's α on the whole set (the primary number).

**Ship criterion.** Written down before the round starts. Common thresholds (Landis and Koch 1977, widely cited but a rough guide — the field varies, and prevalence-adjusted alternatives exist; see Chapter 4):

- κ < 0.40 — "fair or below." Guideline is not ready; another pilot revision.
- κ 0.40–0.60 — "moderate." Marginal for a research eval; not typically acceptable for a production regression gate.
- κ 0.60–0.80 — "substantial." Reasonable for most production settings.
- κ > 0.80 — "almost perfect." Realistic on well-designed tasks with 2–3 anchored labels; ambitious on tasks with more categories or subjective ordinal scales.

Pick the threshold from the *use* of the labels, not from the ease of hitting a nice number. A regression alerting metric can tolerate κ ≈ 0.5 (the noise is amortized across many observations). A safety-critical launch gate cannot; κ ≥ 0.7 is a defensible floor there. A compliance audit typically wants κ ≥ 0.8.

## Making calibration converge

If the pilot revised guideline still produces κ below your threshold in the calibration round, you have three levers, in roughly increasing cost:

1. **Retrain annotators against the revised guideline.** If the guideline changed materially since the pilot, some annotators may still be operating on their older mental model. A short retraining session with the new worked examples and edge cases often recovers 0.05–0.10 of κ.

2. **Simplify the taxonomy.** Fewer labels — especially collapsing an ordinal 5-point into a 3-point — reliably raises κ. This is a tradeoff: you lose resolution in exchange for reliability. It is almost always the correct move on tasks where 5-point κ sits around 0.4; better to have a reliable 3-point metric than an unreliable 5-point one.

3. **Change the pool.** If a specific annotator is consistently disagreeing with the rest and their per-annotator quiz score was borderline, replace them. If a broad crowd platform is producing κ ≈ 0.35 on a task that a small in-house team achieves κ ≈ 0.7 on, the task is probably too specialized for crowd work (Chapter 7's build-vs-buy question).

The lever *not* to pull is "run the study anyway and hope the aggregate averages out." Bad-κ labels produce bad calibrations, bad training signals, and bad leaderboards, and each downstream user of the labels amplifies the noise rather than averaging it out.

## The one-page pilot checklist

A concrete artifact you can copy into a project template:

```
# Pilot Round Checklist — <task>

## Before the pilot
- [ ] guidelines_v0.md written and committed
- [ ] Qualification quiz built (10-20 items, target ≥ 80% pass)
- [ ] Pilot sample assembled (50-200 items, edge-weighted)
- [ ] Every pilot item to be labelled by every pilot annotator
- [ ] Labelling tool project set up, dry-run by one team member
- [ ] Ship threshold for the *calibration* round written down

## During the pilot
- [ ] Annotators trained on guidelines_v0 (session or async video)
- [ ] Feedback channel open (Slack, form, tool-native)
- [ ] Time-per-item logged

## After the pilot
- [ ] Per-item agreement computed and ranked
- [ ] 2-hour synchronous review of top 20-30 disagreeing items
- [ ] guidelines_v1.md with explicit changelog
- [ ] Any new worked examples added
- [ ] Retrain annotators on v1 changes
- [ ] Pilot κ, per-item time, and estimated per-label cost recorded

## Calibration round
- [ ] Representative sample (100-300 items)
- [ ] Partial overlap (every item ≥ 2 annotators; every annotator pair overlaps)
- [ ] Ship threshold met with 95% CI clearly above threshold
  (bootstrap 1,000+ resamples; report the interval, not just the point)
```

Every one of those checkboxes is something a real project has skipped at some point and paid for later. Turning it into a checklist is how you cheaply avoid that pattern.

## Where the pilot fits in the larger loop

The pilot / calibration / production cycle is not linear. In practice:

- **Cycle 1:** Pilot on a v0 guideline → revise to v1 → calibration passes → production batch begins → over the first 500–1,000 items, new edge cases emerge → open the guideline for a *minor* revision to v1.1 (do not restart the pilot for minor revisions, but do log the change).
- **Cycle 2 (task expansion):** A new subdomain enters scope (new topic, new language, new response format). Draw a *targeted* pilot from that subdomain, re-run the pilot + calibration for it, decide whether the existing guideline extends or whether the subdomain gets its own guideline branch.
- **Cycle 3 (annotator turnover):** New annotators onboarded. They go through the qualifier and a short calibration set to verify they align with the rest of the pool. Their first few hundred items get elevated review. If a new annotator's κ against the existing pool is < 0.5, they do not enter the production queue.

The versioned changelog on the guideline is what makes this loop auditable. When a reviewer six months later asks "which guideline version was in effect when this label was produced," you should be able to answer without archaeology.

## Summary

A pilot round exists to find and fix the ambiguity in a v0 annotation guideline before it costs a full batch of labels. It uses an edge-weighted 50–200 item sample labelled by every prospective annotator, produces a ranked list of disagreements, and drives a synchronous revision session that yields v1 of the guideline. The calibration round then verifies on a representative sample that v1 achieves the pre-committed agreement threshold, using a partial-overlap structure so that both pairwise and multi-rater agreement statistics are estimable. When calibration falls short, the levers in cost order are annotator retraining, taxonomy simplification, and pool changes — never running production anyway. A one-page checklist and a versioned guideline changelog turn this from process discipline into cheap habit. The next chapter is the agreement-statistic math the whole loop turns on.
