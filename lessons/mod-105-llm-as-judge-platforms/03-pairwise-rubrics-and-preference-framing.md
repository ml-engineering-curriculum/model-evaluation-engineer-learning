# Pairwise Rubrics And Preference Framing

Pairwise judging drops the fixed scale. Instead of scoring one response, the judge sees two responses to the same prompt and picks a winner (or declares a tie). The rubric moves from "how good is this" to "which is better and why." The mechanical shape is different, the failure modes are different, and — the reason this pattern took over model ranking — the *relative* judgment is genuinely more robust to rubric ambiguity than the absolute one. This chapter is about designing pairwise rubrics that stay honest, and about the specific gotchas that only show up when you switch from single-answer to head-to-head.

## Why pairwise is often better for ranking

The core insight — the one that made Chatbot Arena and MT-Bench's pairwise mode viable — is that people (and judges) are much better at *comparing* two responses than at *scoring* one in isolation. Absolute scoring requires the judge to hold a stable internal scale ("what does a 4 mean today, and did it mean the same thing last week?"). Pairwise scoring only requires the judge to say "A is better than B on this criterion." Even a judge with drifting internal calibration will typically pick the same winner across runs on the same pair.

The practical consequence is that pairwise rubrics are usually shorter and easier to write than absolute ones. They still need anchors — but the anchors are about *what dimensions of the comparison matter*, not about score points. The three properties every pairwise rubric should have:

- **Explicit criteria** for what "better" means (correctness, helpfulness, format compliance, safety), so the judge is not inventing them.
- **A bounded verdict shape** — typically `{A, B, tie}`, sometimes `{A>>B, A>B, tie, B>A, B>>B}` for a graded preference.
- **Position-neutral wording** — the two responses are called by neutral labels (A and B, or Response 1 and Response 2), not by model name.

Pairwise scoring pays for that robustness in two currencies. First, it gives up per-item interpretability — you no longer have a per-example score, only a per-pair winner, which makes slicing harder. Second, it inherits a specific systematic bias — *position bias*, the judge preferring whichever response appeared first — that absolute rubrics do not have. Chapter 4 covers the swap-and-average control for that; this chapter just flags it as a design constraint that the rubric prompt itself must not amplify.

## A concrete pairwise rubric

Same customer-support setup as Chapter 2, now in pairwise form.

```
You are an expert grader comparing two customer-support assistant replies
to the same user question.

You must decide which reply is better against the following criteria, in
order of priority:
  1. Correctness — the reply's factual claims are supported by the
     reference below and do not contradict it.
  2. Coverage — the reply addresses the user's actual question.
  3. Scope — the reply does not add advice or claims outside the reference.

If both replies are similar quality on all three criteria, output tie. Do
not use "tie" as a hedge; only when they are genuinely close.

User question:
"""
{question}
"""

Reference answer (from product documentation):
"""
{reference}
"""

Reply A:
"""
{reply_a}
"""

Reply B:
"""
{reply_b}
"""

First, in 2–4 sentences, reason about how the two replies compare on each
of the three criteria, in the order given. Then, on a final line by itself,
output exactly one of:

  Verdict: A
  Verdict: B
  Verdict: tie
```

Same six rules as the absolute rubric — explicit criteria, reference in the prompt, CoT-before-verdict, parseable final line — reshaped for the comparison case. The order-of-priority hint (correctness before coverage before scope) is doing real work: it prevents the judge from producing "A is more thorough but B is more accurate" and then flipping a coin. Priority-ordered criteria give a deterministic tie-breaking rule the rubric can point to.

## Verdict shapes: binary, ternary, and graded

Three common verdict spaces, in order of when to use each.

**Binary (A or B, no tie).** Simplest to aggregate — every pair produces a winner. Works when the task is asymmetric enough that genuine ties are rare (e.g., safety compliance: either a reply is safe or it isn't). Forces the judge off the fence, which is sometimes what you want and sometimes what makes the number less honest.

**Ternary (A, B, tie).** The most common shape. Adds a tie option, at the cost of a modeling decision downstream: how do Bradley-Terry / ELO systems handle ties? Two standard conventions — treat a tie as 0.5 wins for each side (the ELO convention), or drop tied pairs from the fit (some Bradley-Terry variants). Pick one, document it, and be consistent. Chapter 6 discusses the statistical impact.

**Graded (A>>B, A>B, tie, B>A, B>>B).** Adds intensity to the preference. Useful when downstream you care about *how much* better one system is, not just whether it is better. Costs more judge tokens (the rubric has to define what >> means vs >), more modeling complexity, and is only worth using if you actually consume the intensity signal downstream. If your aggregation collapses graded to binary anyway, don't bother.

Default to ternary. Move to binary if ties are genuinely rare on your task. Move to graded only if a downstream consumer (a ranking system, a reward-model training pipeline) actually uses the intensity.

## Position bias is a rubric-design concern, not just an aggregation concern

Chapter 4 covers the *aggregation-side* fix for position bias (swap-and-average). But the *rubric-side* half — not making it worse — is a Chapter 3 responsibility. Two design tactics:

**Use neutral, symmetric labels.** "Response A" and "Response B" are symmetric. "The first response" and "the second response" biases the judge toward the first through ordinal salience. "The reference response" and "the candidate response" biases the judge toward the reference through the anchor word. Neutral is better.

**Do not reveal model identity to the judge.** If a rubric says "Compare GPT-4's response to Llama-3's response," the judge's priors on the two models will contaminate the verdict — this is one of the paths self-preference bias takes (Chapter 4 discusses the effect measured in Panickssery et al. 2024). Strip provenance from what the judge sees. Model identity lives in your logging metadata, not in the rubric prompt.

Neither of these fixes position bias by itself. Swap-and-average is still required. But a rubric that names one side "Response 1 (baseline)" and the other "Response 2 (candidate)" makes swap-and-average less effective than it should be, because the labels themselves are asymmetric.

## Pairwise-only failure modes

Two failure modes exist in pairwise rubrics that do not exist in absolute ones.

**Superficial tie-breaking.** If the two responses are genuinely close on the stated criteria, the judge may fall back on style, length, or formality to break the tie. This is the source of length bias (Chapter 4): the longer response is not actually better, but the judge picks it because there is nothing else to distinguish them. Mitigations:
- Explicit tie option (so the judge has a legitimate exit from a coin flip).
- Priority-ordered criteria (so tie-breaking is deterministic and rubric-driven, not aesthetic).
- Length normalisation in the rubric prompt itself: "Do not prefer a longer response merely because it is longer; prefer the one that better addresses the criteria above." Zheng et al. 2023 report this line measurably reduces length bias, though it does not eliminate it.

**Comparison drift on very different responses.** When A and B differ dramatically in style, format, or approach — a bullet-point checklist vs. a prose paragraph, a formal reply vs. a casual one — the judge often gets pulled toward evaluating the *style* rather than the *content*. Mitigation is again in the rubric: state that the format is not part of the criteria unless format compliance is one of them, and give an example of two stylistically different responses judged on content alone.

Both of these are things you can find by pilot-testing the rubric on 5–10 hand-picked pairs (two clear wins for A, two clear wins for B, two genuine ties, two "stylistically very different") before scaling. If the judge gets the clear cases right and its reasoning on the close cases is defensible, the rubric is worth running at scale.

## Parsing the verdict

Same discipline as Chapter 2, adapted for the ternary label space.

```python
import re

VERDICT = re.compile(r"^Verdict:\s*(A|B|tie)\s*$", re.MULTILINE | re.IGNORECASE)

def parse_pairwise(text: str) -> str | None:
    match = VERDICT.search(text)
    if not match:
        return None
    verdict = match.group(1).lower()
    return verdict if verdict in {"a", "b", "tie"} else None
```

Parse failures are logged, counted, and either re-judged or excluded from the aggregate with the exclusion rate reported. A rubric with a > 2% parse-failure rate has a prompt problem — usually the CoT is running long enough that the "Verdict: X" line is not reliably last — and needs a wording fix, not a more forgiving parser.

## Where pairwise rubrics live in the tooling

- **OpenAI evals** supports pairwise rubrics via `evals.elsuite.modelgraded.classify:ModelBasedClassify` with a two-response prompt template, or via the higher-level `evals.elsuite.pairwise` scaffolds.
- **`lm-eval-harness`** has `llm_judge` output-type support for judge models but is weaker on pairwise-specific plumbing; most pairwise work in the harness is written as a custom task.
- **Inspect** provides `model_graded_qa` for absolute rubrics and building blocks (`Solver`, custom `Scorer`) for writing pairwise rubrics as tasks.
- **Prometheus** ships an explicit `Absolute Grading` and `Relative Grading` (pairwise) template pair in its evaluation prompts, with the graded prompt matching the "A / B / tie" shape from this chapter.
- **Chatbot Arena** is the reference implementation of pairwise judging at scale — crowdsourced human pairs plus a GPT-4 judge as a proxy — and its rubric template is a good study.

The specific tooling matters less than the shape. Once you can write a pairwise rubric that follows the six rules from Chapter 2 (restated for comparison), the platform is a matter of vocabulary.

## Summary

Pairwise judging asks the judge to pick between two responses to the same prompt rather than score one. This is more robust to rubric ambiguity and to the judge's internal calibration drift, at the cost of per-item interpretability and a new systematic bias (position bias). A well-designed pairwise rubric states its criteria explicitly and in priority order, uses neutral symmetric labels, hides model identity from the judge, offers a real tie option, and emits a bounded verdict in a parseable shape. Two pairwise-only failure modes — superficial tie-breaking that leaks into length bias, and comparison drift on stylistically different pairs — are largely rubric-design problems and largely fixable in the prompt. The next chapter takes the biases the rubric cannot fully fix — position, length, self-preference — and shows the aggregation-side controls that keep them from contaminating the aggregate.
