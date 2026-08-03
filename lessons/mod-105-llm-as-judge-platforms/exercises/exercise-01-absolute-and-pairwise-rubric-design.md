# exercise-01: Absolute And Pairwise Rubric Design

**Estimated effort:** 3 hours

## Objective

Design and pilot a *matched pair* of rubrics — one absolute and one pairwise — for the same real product task. Both rubrics must follow the six rules from Chapters 2 and 3 (anchored scoring, explicit criteria, reference included, CoT-before-verdict, bounded parseable output, versioned artifact). Then pilot both against 20–30 hand-picked items and produce a rubric v1 that is ready to run at scale. The deliverable is not a working eval; it is a *pair of rubric artifacts* you have already stress-tested against known-answer items.

## Prerequisites

- mod-105 Chapters 1, 2, 3 (LLM-as-judge modes, absolute rubric design, pairwise rubric design).
- mod-102 Chapter 4 (inter-annotator agreement, adjudication) — useful for the pilot phase.
- Access to at least one capable judge model (frontier hosted, or a strong OSS instruction-tuned model). This exercise is about rubric design, not judge selection; a single judge is enough.

## The task: pick a real product-shaped one

Choose a task where model-graded evaluation is genuinely necessary — exact match is not sensible — and where a reference answer exists (or can be written) for each item. Good candidates:

- **Customer-support triage.** Given a user question and reference documentation, is the assistant's reply correct, complete, and in-scope?
- **Documentation-grounded Q&A.** Given a passage and a question, is the assistant's answer supported by the passage?
- **Code-review comment quality.** Given a diff and a review comment, is the comment on-point, technically correct, and actionable?
- **Summarization faithfulness.** Given an article, is the summary faithful (no hallucinated claims) and complete on the key points?
- **Format-compliant generation.** Given a set of formatting constraints, does the output satisfy them?
- **Your own domain.** As long as items and references are legitimately shareable and the rubric criteria are defensible.

Constraints on the task:

- **Genuine ambiguity in the middle.** If every response is trivially "correct" or "incorrect," you do not need model-graded scoring and this exercise is uninteresting.
- **Write-down-able reference.** A rubric that requires the judge to judge from scratch is much less reliable (Chapter 2 rule 4). If your task has no reference, constrain the rubric criteria to properties the judge can check without one (length, format, refusal).
- **A shared task across the two rubrics.** The pairwise rubric compares responses to the same prompts the absolute rubric scores. Do not use two different task definitions.

## Requirements

### Part A — the absolute rubric

Ship `rubrics/absolute_v1.md` containing:

1. **Header.** Task name, version (v1), date, one-line task description, and expected downstream use ("regression alerting on release," "release gating," "reward signal for training," etc.).
2. **Criteria block.** A short explicit statement of the criteria being scored (correctness / coverage / scope, or whatever your task requires). One sentence per criterion.
3. **Scoring shape choice.** State whether you are using binary, small-ordinal (3–5 categories), or multi-criterion, and briefly justify against the Chapter 2 shape-selection rules.
4. **Anchor block.** One paragraph per score point, describing when the response is at that point in terms of *properties of the response*, not the reader's reaction.
5. **Prompt template.** The full rubric prompt including `{question}`, `{reference}` (if applicable), `{reply}` slots and the "reason first, then final line" pattern. The final line must be regex-parseable to a single label (see Part D).
6. **Change-log stub.** A one-line entry: "v1: initial." Future versions will accumulate here.

### Part B — the pairwise rubric

Ship `rubrics/pairwise_v1.md` containing:

1. **Header.** As Part A.
2. **Criteria block, priority-ordered.** Same criteria as the absolute rubric, in an explicit priority order so tie-breaking is deterministic (Chapter 3).
3. **Verdict shape choice.** Binary (A/B), ternary (A/B/tie), or graded (A>>B, A>B, tie, B>A, B>>A). Justify against the Chapter 3 shape-selection rules; ternary is the default.
4. **Prompt template.** Full prompt with `{question}`, `{reference}`, `{reply_a}`, `{reply_b}` slots, an anti-length line, and the "reason first, then final line" pattern. Neutral labels (Response A / Response B), no model identity.
5. **Change-log stub.** "v1: initial."

### Part C — the pilot item set

Ship `pilot/items.jsonl` with 20–30 items, one JSON object per line:

```
{
  "id": "pilot-001",
  "question": "...",
  "reference": "...",
  "reply": "...",
  "reply_alt": "...",           # a second response for the pairwise pilot
  "expected_absolute": 4,       # your considered judgment as a domain rater
  "expected_pairwise": "A"      # A / B / tie for (reply=A, reply_alt=B)
}
```

The pilot set must include, at minimum:

- 4 items where the correct absolute score is clearly the top of the scale (the response is unambiguously right).
- 4 items where the correct absolute score is clearly the bottom (unambiguously wrong).
- 4 items in the middle band where reasonable raters could differ by one point.
- 4 pairs where response A is unambiguously better than B (or vice versa).
- 4 pairs that are genuine ties on the stated criteria.
- 4 pairs where A and B are stylistically very different but similar quality (this stresses the pairwise rubric against style bleed).

You may reuse the same base items across absolute and pairwise where sensible.

### Part D — parsing

Ship `parse.py` with two functions:

```python
def parse_absolute(judge_text: str) -> int | None: ...
def parse_pairwise(judge_text: str) -> str | None: ...   # returns "A" | "B" | "tie" | None
```

Both are regex-based, deterministic, and return `None` on parse failure. Test cases in `test_parse.py` cover: well-formed outputs at every label, outputs with trailing whitespace, outputs where the CoT is long, and pathological outputs where the judge disobeyed the format (the parser must return `None`, not silently guess).

### Part E — run the pilot

Run each rubric against the pilot set through one judge model. Record for each item:

- Judge raw output.
- Parsed verdict (or `None`).
- Expected verdict from `items.jsonl`.
- Agreement flag (judge matches expected).

Aggregate:

- **Absolute:** per-score confusion (expected vs. judge), overall accuracy, off-by-one rate on ordinal rubrics.
- **Pairwise:** overall accuracy, tie rate (judge vs. expected), swap-inconsistency rate (you do not have to run the full swap yet, but re-run the "genuine tie" and "close pairs" subset in the swapped order and flag flips).

### Part F — the report

Write `REPORT.md` (≤ 2 pages) covering:

1. **Task definition.** What you're grading and why exact match is insufficient.
2. **Shape decisions.** Why binary / small-ordinal / graded for each rubric.
3. **Pilot results.** Both aggregates plus the confusion tables.
4. **Rubric flaws found.** For each pilot miss, name the specific rubric flaw (anchor ambiguity, missing criterion, reference asymmetry, etc.) and how you would fix it in v2. This is the most important section — the point of the pilot is to *find the flaws*, and a report where every pilot item passed suggests a pilot set that was not stressful enough.
5. **Ship / rework decision.** Is v1 ready to run at scale (100+ items with a κ measurement)? If not, what specific edits are queued for v2?

### Part G — bundle

- `rubrics/absolute_v1.md`, `rubrics/pairwise_v1.md`
- `pilot/items.jsonl`
- `parse.py`, `test_parse.py`
- `logs/absolute_pilot.jsonl`, `logs/pairwise_pilot.jsonl`
- `REPORT.md`
- `run_pilot.sh` — one-shot rerun of both pilots

## Starter guidance

- **Write the pilot items before the rubric.** If you write the rubric first, you'll unconsciously construct items that vindicate the anchors you already wrote. Draft the items — especially the ambiguous middle band — from real product data or plausible near-real data, *then* write the rubric to handle them.
- **A pilot where the judge got every item right is a bad pilot.** It means your items were too easy, or your rubric let the judge off the hook. Rework the item set until you find at least 2–3 items where the judge is wrong for an *identifiable* reason. Those items are the design signal.
- **Use exactly the same criteria wording across the absolute and pairwise rubrics.** If "correctness" means different things in the two prompts, the two rubrics are measuring different constructs and any downstream comparison is meaningless.
- **CoT-before-verdict is worth the tokens.** Every rubric in this exercise should ask the judge to reason first. If you skip this to save cost, do not be surprised when the judge produces less consistent verdicts.
- **The final line must survive a chatty judge.** Test the parser on outputs where the judge writes a long CoT, then a sentence like "So the final verdict is B" that is *not* your target format. The parser should return `None` on those and the judge prompt should be tightened until they don't happen.
- **Do not use the same model as subject and judge.** Even in the pilot, this contaminates the exercise with self-preference (Chapter 4). Use a different model to generate the pilot responses than the model doing the judging.
- **Anchors are not "definitions in a dictionary."** They should read like "the response would be a 4 if..." — an actionable check a rater can apply. If an anchor reads like a definition of the score word ("a 4 is a very good answer"), rewrite it as a property test.
- **Pairwise ties are load-bearing.** Rubrics that force a winner on genuine ties push the judge into arbitrary tie-breaking, which is where length bias enters. A rubric with a real tie option and a "do not use tie as a hedge" instruction is much cleaner than a binary rubric.
- **You are not writing the solution.** The paired solutions repo will demonstrate a full grading pipeline; this exercise is about producing *the rubric artifacts* and validating them on a pilot. Do not run 500 items yet — exercise 03 is the calibration study at that scale.

## Acceptance criteria

- Two rubric artifacts (`absolute_v1.md`, `pairwise_v1.md`) are shipped and each satisfies the six-rule checklist from Chapters 2 and 3 (anchored / criteria explicit / reference included where applicable / CoT-before-verdict / parseable bounded output / versioned with change-log).
- The pilot item set has at least 20 items covering the six required categories (top-anchor / bottom-anchor / ambiguous-middle / clear-winner / genuine-tie / style-different).
- Both rubrics have been run against the pilot with a single judge and per-item results are logged.
- The `REPORT.md` names at least two specific rubric flaws surfaced by the pilot, with concrete v2 fixes.
- A reviewer with judge credentials can rerun both pilots via `run_pilot.sh` and reproduce the aggregate.
- Parser test cases include at least one adversarial output that the parser correctly rejects (`None`), and this is tested via `test_parse.py`.

## Stretch goals

- **Rubric wording ablation.** Ship two variants of the absolute rubric — a stricter and a more lenient wording of the anchors — and rerun the pilot under each. Report the delta in mean score and per-item agreement. Rubric wording is often the largest source of noise; this is how you measure it.
- **Two-judge pilot.** Run the pilot with a second judge from a different model family and compare the two per-item. Any item where the two judges disagree is a candidate for a rubric edit (the criteria are underspecified) or a task-level flag (the item is genuinely ambiguous).
- **Adversarial pilot items.** Add 4–8 items designed to trigger specific known biases: two responses of very different length but equivalent content (length bias probe), a response written in the judge model's own style (self-preference probe), a response where A and B are near-identical (position bias probe). These items should ideally *not* fool your rubric; when they do, it is a v2 design signal.
- **Rubric change-log discipline.** Even at v1, write the change-log as if a v2 will follow. State what future rubric edits would prompt a version bump and how their numeric impact would be measured. This is the discipline that keeps a rubric-based dashboard honest across releases.
