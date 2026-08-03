# Absolute Rubrics With Anchors

An absolute rubric asks the judge one question about one response: on a fixed scale, how good is it against these criteria? The naive version — "rate this response from 1 to 10 for helpfulness" — is where most first attempts start and where most of the reproducibility problems come from. This chapter is about the specific properties that turn an absolute rubric from "vibes on a number line" into a measurement whose per-item scores mean the same thing across judges, releases, and dataset revs.

## What "anchor" means

A rubric *anchor* is a fixed description bound to a specific score point on the scale. Instead of "rate from 1 to 5," an anchored rubric says:

```
5 — Fully addresses the user's question with no factual errors and follows all constraints.
4 — Addresses the question with minor factual issues OR misses one non-critical constraint.
3 — Partially addresses the question; missing significant content or has a moderate factual error.
2 — Off-target OR contains a critical factual error that would mislead the user.
1 — Does not attempt the task, refuses inappropriately, or produces nonsense.
```

Every score point is tied to a description that a rater — human or model — can check against. The rubric now defines what "3" means. Two independent judges are much likelier to converge; the same judge across runs is much more consistent; and — crucially — you can *audit* an item where the judge produced a 4 by asking whether the description in the "4" anchor actually holds for that response. Without anchors, there is nothing to audit against.

The anchoring literature has a long history in human survey design (behaviorally-anchored rating scales, BARS, in industrial-organizational psychology going back to Smith and Kendall 1963) and it maps cleanly onto LLM judges. Zheng et al. 2023 report that adding score anchors to the MT-Bench rubric measurably tightens agreement between GPT-4 and human raters relative to an unanchored 1–10 scale, and the same effect is why every serious model-graded rubric shipped since — OpenAI evals' `cot_classify`, Prometheus's rubric templates, Inspect's `model_graded_qa` examples — is anchored, not free-form numeric.

## Choose the scoring shape deliberately

Before you write anchor text, pick the scoring shape. There are three defensible ones, and each has a specific use case.

**Binary (Yes / No).** The rubric asks whether a single criterion holds. "Did the response follow all formatting constraints?" "Is the response free of unsupported factual claims?" Aggregation is a fraction. Binary rubrics are the easiest to calibrate against humans (the label space is small enough that κ is stable on modest gold sets), and they are the shape you want for behavior gates ("no shipped release with < 95% refusal-compliance on this red-team subset").

**Small ordinal (3–5 categories).** The rubric asks for one of a small set of ordered categories: A/B/C, or 1–5. Aggregation is a mean, or a per-category rate. This is the sweet spot for most quality rubrics — enough resolution to distinguish "good" from "great" and "bad" from "broken," few enough categories that a rater can hold all the anchors in mind at once. Anything wider than about 5 categories starts to lose inter-rater reliability without buying you real resolution.

**Multi-criterion score.** Instead of one score, the judge produces a small vector — helpfulness, harmlessness, honesty, format-compliance — each on its own small-ordinal scale, then reported per-dimension. This is the shape Prometheus uses. The aggregation is per-dimension; only combine into one number if the downstream decision genuinely requires a scalar (in which case, weight the components deliberately and document the weights).

Avoid: unanchored numeric scales wider than about 5 points ("1–10" with no per-score description). The judge will use them inconsistently, humans will use them differently from the judge, and κ against humans will be low for a reason that has nothing to do with judge quality.

## The anchoring rules that actually matter

The following rules are the ones that hold up across MT-Bench-style, Prometheus-style, and OpenAI-evals-style rubrics. Each has a failure mode you can point at when it is skipped.

**Rule 1: Every score point has a distinct, testable description.** A rater has to be able to say "this response matches anchor 4 and not anchor 3." "Somewhat good" and "quite good" are not distinct. "Addresses the question with minor factual issues" and "partially addresses the question with a moderate factual error" are — one is about severity, one is about coverage. If two anchors are indistinguishable in practice, drop one and renumber.

**Rule 2: Anchors describe the response, not the reader's reaction.** "Score 5 if the answer is impressive" is a reaction. "Score 5 if the answer covers all the reference's key points" is a property of the response. Model judges (and humans) can check the second; the first they will confabulate.

**Rule 3: The rubric includes the criteria explicitly, not just the scale.** A rubric that says "rate helpfulness 1–5" implicitly asks the judge to invent what "helpfulness" means. State the criteria: what counts, what doesn't. Zheng et al. 2023's MT-Bench rubric spells out that a good answer should "be relevant, accurate, and concise," and the version that omits this line has measurably worse human agreement.

**Rule 4: For comparison rubrics, include the reference in the prompt.** If your task has a reference answer, put it in the rubric prompt and frame the rubric around comparison: "how well does the response cover the same content as the reference." Judges asked to score from scratch — no reference — are much noisier than judges asked to check against a fixed target. This is the single largest lever on model-graded rubric quality on any task where a reference exists.

**Rule 5: Ask for reasoning before the verdict.** The rubric prompt should ask the judge to write a short justification *first*, then emit the score on a fixed final line. This is the "chain-of-thought before the verdict" pattern; OpenAI evals encodes it as `eval_type: cot_classify`. On non-trivial rubrics it typically moves grader-vs-human κ up by 5–15 points relative to "answer the score directly" (this is the effect Zheng et al. 2023 documents on MT-Bench, and it replicates on internal rubrics).

**Rule 6: The verdict is emitted in a parseable, bounded shape.** A judge that returns "I'd say a 4 or maybe a 5, it's close" gives you nothing to score. A judge asked to end its response with `Final score: <A|B|C>` gives you a regex-parseable label. Enforce the shape in the prompt, parse it deterministically, and fail loudly (drop-with-log, do not silently coerce to a default) when the parse fails.

## A concrete rubric

Here is an absolute rubric for the running example we will use throughout the module — a customer-support assistant. The task is: given a user question about a product, is the assistant's reply helpful, correct, and appropriately scoped?

```
You are an expert grader for a customer-support assistant.
You will read a user question and the assistant's reply, and rate the reply
against a rubric.

Rubric criteria:
  - Correctness: the reply's factual claims are supported by the reference
    below, and it does not contradict the reference.
  - Coverage: the reply addresses the user's actual question (not a nearby
    question or a canned response).
  - Scope: the reply does not add advice or claims outside the scope of the
    reference (no hallucinated policy, pricing, or steps).

User question:
"""
{question}
"""

Reference answer (ground truth for this question, from the product
documentation):
"""
{reference}
"""

Assistant reply to grade:
"""
{reply}
"""

Score the reply on the following 4-point scale.
Anchors:
  4 — Fully correct and complete: covers the reference's key points, adds
      no unsupported claims, and directly answers the user question.
  3 — Mostly correct: covers most of the reference's key points, may omit
      one non-critical point, and adds no unsupported claims.
  2 — Partially correct: covers some of the reference's key points, OR
      adds a minor unsupported claim, OR partially misses the question.
  1 — Wrong or misleading: contradicts the reference, invents a policy
      or step, refuses inappropriately, or does not attempt the question.

First, in 1–3 sentences, reason about how the reply compares to the
reference against each of the three criteria. Then, on a final line by
itself, output exactly one of:

  Final score: 1
  Final score: 2
  Final score: 3
  Final score: 4
```

The rubric encodes all six rules: distinct anchors, response-focused descriptions, explicit criteria, reference included, CoT-before-verdict, parseable final line. The parser is a single regex.

```python
import re

VERDICT = re.compile(r"^Final score:\s*([1-4])\s*$", re.MULTILINE)

def parse_verdict(text: str) -> int | None:
    match = VERDICT.search(text)
    return int(match.group(1)) if match else None
```

An item where the parse fails is *not* silently defaulted to a middle score. It is logged, counted, and either re-judged (with a slightly reworded prompt) or excluded from the aggregate with the exclusion rate reported.

## The rubric is a versioned artifact

Once a rubric is in use, wording changes to it are not "just prompt engineering." They are equivalent to a metric definition change. A rubric edit that moves the mean score by 0.3 will look identical in downstream dashboards to a real product regression of the same magnitude. Two disciplines keep this from becoming a source of confusion:

- **Version the rubric.** `rubric.v1.md`, `rubric.v2.md`, with a changelog that says what changed and — where measurable — what the numeric impact on the eval set was. Any dashboard that plots a rubric-based score must show the rubric version alongside.
- **Re-score the golden slice on rubric change.** When you bump a rubric version, rerun the judge over a fixed anchor set of items (30–100 stable examples) and publish the old-vs-new score delta. This is analogous to Chapter 5's calibration slice; the rubric change is the "release" being tested.

Skip these two disciplines and you will spend an on-call rotation chasing a "regression" that is actually a rubric wording tweak someone made and forgot to announce.

## What good looks like

At the end of this chapter you should be able to look at any absolute-scoring rubric and answer:

- What scoring shape does it use? Binary, small-ordinal, or multi-criterion?
- Are the score points anchored to distinct, testable descriptions?
- Are the criteria stated explicitly, not left implicit in the scale label?
- Does the rubric include the reference where one exists?
- Does the rubric ask for reasoning before the verdict?
- Is the verdict emitted in a bounded, parseable shape?
- Is the rubric versioned, and is there a change-log entry with a numeric-impact note?

Every "no" on that list is a specific reproducibility risk you can name. Chapter 3 does the same work for the pairwise case, where the questions are similar but not identical.

## Summary

An absolute rubric turns a per-item response into a per-item score against a fixed rubric. The property that makes those scores comparable across judges and across releases is *anchoring* — every score point tied to a distinct, testable description of the response. The scoring shape (binary, small-ordinal, multi-criterion) is a deliberate choice, not a default. The rubric prompt states the criteria explicitly, includes the reference where one exists, asks the judge to reason before emitting the verdict, and enforces a parseable output shape. Rubrics are versioned artifacts; changes to the wording move the mean and must be logged the way a metric-definition change would be. The next chapter takes the same discipline into the pairwise setting, where the judge does not score responses in isolation but picks a winner between two.
