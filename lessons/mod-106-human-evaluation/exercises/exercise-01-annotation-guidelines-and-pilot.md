# exercise-01: Annotation Guidelines and Pilot

**Estimated effort:** 3 hours

## Objective

Author a v0 annotation guideline for a real, non-trivial labelling task, run a 50-item pilot with at least two annotators, and produce a v1 guideline whose changelog explicitly names the edge cases the pilot surfaced. The deliverable is not "a guideline" — it is the *evidence* that the guideline was iterated against real data with real disagreements, packaged so a downstream owner can trust and extend it.

## Prerequisites

- mod-106 Chapters 2 and 3.
- A task where you have (or can produce) 50–200 unlabelled items — chat conversations, LLM responses, summaries, code snippets, product reviews, whatever fits. Real data from a project you work on is ideal; a public dataset (a slice of `alpaca-eval`, `mt-bench`, or any Hugging Face dataset with free-form text) is fine.
- At least 2 human annotators. You can be one. The other can be a teammate, mentor, or friend willing to spend ~60 minutes labelling under a written guideline. Real production evals typically use paid annotators; for the exercise, informal recruitment is fine as long as they follow the written guideline rather than intuition.
- A labelling tool. Options in decreasing order of setup effort: Label Studio, Prodigy, Argilla, a Google Form, a shared Google Sheet with a labels column. The tool matters less than that both annotators use the same one and can't see each other's labels during labelling.

## The task

Pick one task with categorical or ordinal labels. Examples that work well at 50-item scale:

- **Reply helpfulness on a 3-point scale** — labels `helpful`, `partially_helpful`, `not_helpful` on assistant replies to user questions.
- **Summary faithfulness on a 3-point scale** — `faithful`, `partially_faithful`, `unfaithful` on model-generated summaries against source text.
- **Refusal classification** — `refused`, `partially_refused`, `answered` on model responses to a set of possibly-adversarial prompts.
- **Code-suggestion quality** — `correct_and_runs`, `almost_correct`, `wrong` on model code completions.

Avoid picking a binary task (too easy to hit κ = 0.9 for the wrong reasons). Avoid picking a 5+ level ordinal task (too hard to hit any κ at 50-item scale for a first pilot). Three-way categorical or ordinal is the sweet spot.

## Requirements

### Part A — v0 guideline

Write `guidelines_v0.md` following the Chapter 2 five-section structure:

1. **Task in one sentence** — what is being decided, on what unit, from what input, on what scale.
2. **Label definitions with anchoring examples** — for each label: one-sentence definition, 2–5 positive examples, 1+ negative-looking-positive, the axis that separates this label from the adjacent one.
3. **Explicit decision rules for common edge cases** — at least four: what happens on malformed input, partial correctness, refusals, and out-of-scope items you expect to encounter.
4. **Worked examples** — 6–12 fully-labelled items with input, correct label, one-sentence reason.
5. **Non-goals** — at least three things annotators are *not* judging.

Follow the seven writing rules: response-focused wording, mutually exclusive labels, anchored ordinal levels (if ordinal), 3–5 categorical levels, no compound criteria, one page of prose or less (worked examples don't count), versioned with a changelog at the top.

### Part B — pilot sample

Assemble `data/pilot.jsonl` — 50 items drawn *deliberately* to maximize edge-case discovery, not uniformly at random:

- ~15 items you suspect are hard (ambiguous, edge-case, borderline).
- ~10 items sampled from the tails of any easily-computed feature (very short, very long, unusual formatting).
- ~10 items from any known subgroup (topic, response format, language).
- ~15 items you consider "obviously easy" — the sanity anchors.

Each item carries at minimum `id`, `input` (the prompt or source text), `response` (the response to be labelled).

### Part C — run the pilot

Every pilot annotator labels every pilot item. Ship `data/pilot_labels.csv` with columns:

```
id, annotator_a_label, annotator_a_notes, annotator_a_seconds,
    annotator_b_label, annotator_b_notes, annotator_b_seconds
```

Rules:

1. Both annotators have `guidelines_v0.md` open in front of them for the full session.
2. Annotators do not confer on individual items during labelling. If they need to clarify the guideline, that is a v1 change, not an in-session fix.
3. Annotators record time per item (the labelling tool should do this; if not, timestamp each label).
4. Neither annotator sees the other's labels until the pilot is complete.

Compute pilot-round agreement using the statistic appropriate to your task (Cohen's κ for categorical, weighted κ for ordinal — see Chapter 4). Report the point estimate; on a 50-item pilot the CI is going to be wide, and you should note this explicitly.

### Part D — the revision session

Rank the 50 pilot items by disagreement (items where annotators disagreed at the top). For the top 15–20 disagreeing items:

1. Read the item, both labels, both note fields.
2. Decide the correct label under `guidelines_v0.md` as written. If the guideline is unambiguous but an annotator misapplied it, that is a training issue, not a guideline issue.
3. Decide whether the guideline as written *actually* supports the correct label unambiguously. If not, revise the guideline. Add an edge-case rule, tighten a definition, add a worked example.
4. Write the revision in the changelog at the top of `guidelines_v1.md`.

Ship `guidelines_v1.md` and `revision_log.md` — a per-item record of every top-tier disagreement and the corresponding guideline change (or "no change, training issue" with a sentence on why).

### Part E — the report

Write `REPORT.md` (1–2 pages) covering:

1. **Task and dataset.** What is being labelled, where the data came from.
2. **v0 guideline design choices.** Why 3-way rather than 5-way, why these labels, what alternatives you considered.
3. **Pilot annotators.** Backgrounds, whether either is you, total labelling time.
4. **Pilot κ.** Point estimate with the caveat that a 50-item pilot has a wide CI; per-item time distribution.
5. **Top 5 disagreement modes.** Named prose descriptions, not just item IDs. "Annotators disagreed on refusals: annotator A read them as `not_helpful`, annotator B as `helpful` with a note. The v1 guideline reserves `not_helpful` for wrong answers and treats refusals as `helpful` with an explicit `refused:` note."
6. **v1 → production readiness.** Under `guidelines_v1.md`, would you commit to a full calibration round on the target ship threshold? If not, what's still missing?

### Part F — bundle

Ship in the exercise directory:

- `guidelines_v0.md`
- `guidelines_v1.md` (with changelog)
- `revision_log.md`
- `data/pilot.jsonl`
- `data/pilot_labels.csv`
- `analysis/agreement.py` — reproducible computation of pilot κ from `pilot_labels.csv`
- `REPORT.md`

## Starter guidance

- **Do not skip the "one page of prose" constraint.** Your v0 guideline will feel too short; that is correct. The worked examples are where the specificity lives. Long prose sections do not survive contact with real annotators, who scan.
- **The pilot annotators should not include the guideline author on the first pass.** If you wrote the guideline, your labels are contaminated by the intent behind the words rather than the words themselves. If you have to be one annotator, log that fact and try to label as if you'd never seen the guideline before.
- **On items where both annotators agreed but you (the guideline author) think they are wrong**, that is guideline evidence — the guideline is guiding annotators toward the wrong answer. Log these separately in `revision_log.md` under a "silent agreement failures" section.
- **Do not over-revise.** If v1 doubles the length of v0, you have probably added rules that will fight each other in production. Cut back. Prefer to revise a label definition rather than add an edge-case rule; edge-case rules proliferate faster than they resolve ambiguity.
- **The pilot κ is not the ship criterion.** The pilot's output is v1 and the revision log, not the κ number. A pilot with κ = 0.3 that produced a substantially better v1 is more valuable than a pilot with κ = 0.75 that produced no revisions (which usually indicates the pilot sample wasn't edge-weighted enough — see Part B).
- **Time per item matters.** An annotator averaging < 15 seconds per item on a task that requires reading a paragraph is not doing the task. Log this in the report; it is context for reading the pilot κ.
- **Preserve raw data.** Do not overwrite `pilot_labels.csv` after the revision session. Post-revision relabelling is a separate artifact.

## Acceptance criteria

- `guidelines_v0.md` has all five sections and follows all seven writing rules from Chapter 2; the version and changelog are at the top.
- The pilot sample is 50 items and is *deliberately edge-weighted* per Part B (uniform-random samples are rejected).
- Both annotators labelled all 50 items independently under the written v0 guideline; `pilot_labels.csv` has per-annotator labels, notes, and per-item times.
- Pilot κ is computed with the appropriate statistic for the task's label shape and reported with a caveat about the small-sample CI.
- `revision_log.md` covers at least 10 top-tier disagreeing items with the guideline change (or "no change, training issue") documented per item.
- `guidelines_v1.md` includes a changelog at the top summarizing revisions; it remains at or under one page of prose (worked examples don't count).
- `REPORT.md` names at least 5 disagreement modes as prose descriptions and makes an explicit v1 → production-readiness call.
- The bundle is reproducible: `python analysis/agreement.py` on the shipped `pilot_labels.csv` reproduces the reported κ.

## Stretch goals

- **Run a v2 mini-pilot.** After producing v1, run 20 more items (drawn from the same edge-weighted distribution) with the same annotators on the revised guideline. Report v1-κ vs. v0-κ. If v1 didn't improve agreement, the revisions were the wrong revisions and the report should analyze why.
- **Three annotators instead of two.** Extend the analysis to Fleiss' κ or Krippendorff's α (see Chapter 4). Compare pairwise κ per pair (A-B, A-C, B-C) to detect whether one annotator is consistently offset from the rest.
- **Reference-sheet artifact.** In addition to the full guideline, ship a one-page annotator reference sheet (Chapter 2's "reference sheet" pattern) — the compressed working document annotators actually keep open. Have your annotators review it and confirm it matches what they were doing.
- **Cost projection.** Given the pilot's median time per item and a target throughput of 10,000 labels, estimate the annotator hours required for the full batch and project the cost under both a crowd (assume $18/hr) and an in-house ($45/hr fully-loaded) sourcing mode. This is a prep exercise for the Chapter 7 sourcing decision.
