# Writing Annotation Instructions That Survive Contact With Data

An annotation guideline is a specification. It is read by people who did not design the task, who do not have access to the designer, and who will each interpret ambiguous instructions differently. If two annotators reading the guideline apply different labels to the same item, the resulting agreement statistic is not measuring "how hard is this task" — it is measuring how bad the specification is. The way you write the guideline is the single largest lever on the reliability numbers you will report in Chapter 4, and the single largest predictor of whether a pilot round finds fifty edge cases you did not anticipate (Chapter 3) or two.

This chapter is about the *specification* itself, before any annotator has ever seen it. The pilot iteration comes next.

## The five parts of a working guideline

A good annotation guideline has five sections in this order. Skipping any of them shows up as disagreement in the pilot.

**1. Task in one sentence.** State what the annotator is deciding, on what unit, from what input, on what scale. "For each user message and assistant reply, label the reply as `helpful`, `partially_helpful`, or `not_helpful`, based on whether it addresses the user's stated question." One sentence. It is the anchor everyone can hold in their head; every subsequent rule refers back to it.

**2. Label definitions with anchoring examples.** For every label in the taxonomy, give:

- A one-sentence definition ("`helpful`: the reply addresses the user's question with correct, actionable information").
- Two to five positive examples ("this reply is `helpful` because…").
- At least one negative example the label is often confused with ("this reply looks helpful but is actually `partially_helpful` because it addresses only the first of two sub-questions").
- The one axis that separates this label from the neighboring label ("the line between `helpful` and `partially_helpful` is whether the reply addresses *every* sub-question the user asked, not the length or tone of the reply").

The last bullet is the one people skip. It is also the one that determines whether annotators agree on the boundary cases, which are the only cases that move κ.

**3. Explicit decision rules for common edge cases.** Every task has a set of situations the naive rubric does not cover. Write them down before the pilot rather than discovering them halfway through:

- What if the input is malformed (empty, non-English, garbled)?
- What if the reply is partially correct and partially wrong — which side wins?
- What if the reply refuses to answer for a reason the annotator considers legitimate?
- What if the reply is off-topic but well-written, or on-topic but poorly written?
- What if the annotator lacks the domain knowledge to judge (a code snippet in a language they don't know, a medical claim they can't verify)?

For each situation, either give a rule ("if the reply is non-English, label as `not_helpful` regardless of content") or an escape hatch ("label as `skip` with a note; the reviewer will adjudicate"). Do not leave these to annotator judgment. If you do, annotators will each make a different local judgment and the aggregate agreement will suffer.

**4. Worked examples.** Six to twelve fully-labelled examples, each with the input, the correct label, and one to two sentences on why. Half of them should be the "obvious" cases so annotators calibrate on what the easy cases look like. The other half should be *previous edge cases the pilot surfaced* (which you will not have yet on v0; you will after the first pilot). A guideline without worked examples is not a guideline; it is a taxonomy.

**5. Non-goals and out-of-scope items.** State what the annotator is *not* judging. "Do not judge grammar unless it is severe enough to affect comprehension." "Do not judge whether the model's answer is one you personally would have given; judge only whether it addresses the user's question." "Do not attempt to verify factual claims outside the reference material; if the claim requires external verification, mark `uncertain`." Annotators will drift toward judging everything they can see; the non-goals section is what pulls them back.

## The seven writing rules

Independent of the section structure, seven writing choices consistently separate guidelines that produce κ ≥ 0.6 from guidelines that produce κ ≤ 0.4 on the same underlying task.

**Rule 1: Response-focused, not model-focused.** Write "if the reply addresses…" not "if the model correctly addressed…". Annotators are judging one artifact — the response in front of them — not making a claim about the model's abilities in general. This wording change reduces the temptation to reward or punish the model for reasons outside the current item.

**Rule 2: Bounded, mutually exclusive labels.** Every item lands in exactly one label. If your taxonomy has three labels but the annotator can pick two of them for the same item, you have two taxonomies. Split them into two questions. A single item with a single label is the only shape that supports κ and Krippendorff's α (Chapter 4). If you need multi-label, split into N binary questions.

**Rule 3: Anchor the ordinal levels.** For ordinal scales (1–5, Likert), every level needs a written anchor. "5 = perfect: reply fully addresses the question with no factual errors, appropriate tone, and no missing information." Not just "5 = excellent." Unanchored ordinal scales drift within an annotator across a session (fatigue moves the middle of their scale by the end of the day) and across annotators (one person's 4 is another's 3). Anchoring is the fix. Weighted κ with quadratic weights (Chapter 4) rewards this consistency; unanchored scales cannot achieve it.

**Rule 4: Prefer three or five categorical levels over two.** Binary labels ("helpful / not helpful") force the annotator to collapse the middle into one of two extremes, and each annotator collapses differently. A three-level scale ("helpful / partially_helpful / not_helpful") gives the middle a home. Five levels give more resolution but require anchoring effort proportional to the number of levels; three is a good default for most tasks. Above seven, agreement drops sharply.

**Rule 5: No compound criteria in a single label.** "Label as `unsafe` if the reply contains harmful content OR is factually wrong OR fails to follow the format." That is three tasks. Split them. When you compound criteria, the annotator picks the criterion that jumps out first and the label distribution reflects that ordering, not the actual rate of any single failure. Split, count each separately, aggregate downstream.

**Rule 6: One page of prose per task, or less.** A guideline that runs ten pages will not be read cover-to-cover by every annotator. Two-page guidelines are read; ten-page guidelines are scanned. Cut ruthlessly and move overflow into worked examples. If the task genuinely needs ten pages of prose, the task probably needs to be decomposed into multiple simpler tasks.

**Rule 7: Version the guideline.** `guidelines_v1.md`, `guidelines_v2.md` — with a changelog at the top of each version. Every pilot round produces at least a v+1. Annotators are trained on a specific version; agreement numbers are reported against a specific version. When the guideline changes, the old κ numbers are no longer directly comparable and the rework required to re-establish them is on the eval owner. This is why guideline changes late in a project are so expensive; underestimating that cost is how projects run over.

## What to include beyond the guideline itself

The guideline document is one artifact in a training package that includes:

- **A qualification quiz** — 10–20 items with known-correct labels that a candidate annotator must label at ≥ 80% accuracy before they enter the queue. This is your first filter; it removes annotators who cannot read the guideline as much as those who cannot do the task.
- **A training set of worked examples** — 20–50 pre-labelled items with the correct label and a written rationale. Candidate annotators go through these before the qualifier, and any team member can point to them when explaining a disagreement.
- **A reference sheet** — a one-page cheat-sheet the annotator has open beside the tool, with the label definitions and the two or three most common edge-case rules. If the guideline is the specification, the reference sheet is the runtime.
- **A feedback channel** — an explicit way for annotators to flag "the guideline does not cover this item." Slack channel, form, tool-native comments. Without this channel, edge cases become silent inconsistency in the labels. With it, they become the next revision.

## Two failure modes to name explicitly

Two patterns burn every human-eval project that has not run one before.

**The over-specified guideline.** The designer, having watched annotators diverge in an early pilot, responds by adding rules — five pages become fifteen, edge cases proliferate, the guideline becomes internally inconsistent. Downstream, agreement often drops rather than rising, because annotators now have to search a long document for the applicable rule and each searches slightly differently. The fix is almost always to *reduce* the taxonomy (fewer, clearer labels), not to add more rules to a taxonomy that is already fighting itself. Sometimes the task is genuinely too hard for the labels you chose.

**The under-specified guideline.** The designer writes one paragraph, sends it to a crowd, and reports the resulting κ as if it measured the task. The κ is bad, of course. The designer concludes the task is intrinsically hard. In reality, the task is not being measured; the annotators are each running their own private task. The fix is a written specification with the five sections above. This is the mode that Chapter 3's pilot procedure is designed to catch.

## What a v0 guideline looks like

A v0 guideline for a "reply helpfulness" task might land at:

```
# Reply Helpfulness Guidelines (v0.1)

## Task
For each (user_message, assistant_reply) pair, label the reply as
`helpful`, `partially_helpful`, or `not_helpful` based on whether it
addresses the user's stated question.

## Labels
- `helpful`: the reply directly addresses every sub-question the user
  asked, with correct, actionable information.
  Positive: "How do I set the timeout in requests?" -> "Pass timeout=5
  to requests.get(...). The value is in seconds."
  Negative-looking-positive: reply is long and correct but the answer
  is buried in the third paragraph; still `helpful`.

- `partially_helpful`: the reply addresses some but not all
  sub-questions, OR gets the main answer right but includes a
  significant incorrect statement.
  Positive: user asks "how do I set timeout AND enable retries";
  reply covers timeout but not retries.

- `not_helpful`: the reply does not address the question, or the
  answer is wrong in a way that would mislead the user.
  Positive: user asks about `requests` in Python; reply describes
  `axios` in JavaScript.

## Boundary rule
The line between `helpful` and `partially_helpful` is *coverage of
sub-questions*, not tone, length, or formatting. A brief correct
answer to a single-part question is `helpful`; a long correct answer
to only one of two sub-questions is `partially_helpful`.

## Edge cases
- If the reply refuses to answer, label `not_helpful` with a note
  explaining the refusal (`refused: safety`, `refused: unknown`).
- If the reply is in a different language than the user's message,
  label `not_helpful`.
- If you cannot verify a factual claim (medical, legal, jurisdiction-
  specific), label `uncertain` with the specific claim in the note.

## Non-goals
- Do not judge grammar or style.
- Do not judge whether the reply is one you personally would have
  given.
- Do not attempt web lookups to verify claims; use the `uncertain`
  label instead.

## Worked examples
[6-12 examples with (input, label, one-sentence why)]
```

That is a v0. It will be wrong in ways the pilot will expose. It is *specific* in ways that let a pilot expose them. A v0 that is vague ("rate the response for helpfulness") does not admit disagreement because there is no rule to disagree about — every annotator uses their own private rule, and the pilot round teaches you nothing you can act on.

## Summary

An annotation guideline is a specification whose readers cannot ask you follow-up questions. It has five sections — task-in-one-sentence, label definitions with examples, edge-case rules, worked examples, and non-goals — and seven writing rules that consistently separate specifications that produce κ ≥ 0.6 from those that produce κ ≤ 0.4: response-focused wording, bounded exclusive labels, anchored ordinal levels, three-to-five categorical levels, no compound criteria, one page or less of prose, and explicit versioning. It ships alongside a qualification quiz, a training example set, a one-page reference sheet, and a feedback channel. The two failure modes to name explicitly are the over-specified guideline that hurts agreement by making the specification internally inconsistent and the under-specified guideline whose bad κ is measuring the writing quality, not the task. The next chapter runs the pilot round that turns v0 into a version you can actually deploy.
