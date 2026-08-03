# Bias Controls: Position, Length, And Self-Preference

Judge models have systematic biases. Three of them are well-measured, cheap to control, and — if uncontrolled — large enough to reverse a system ranking or move an absolute score by several percentage points. Any team that ships an LLM-graded number without at least these three controls is publishing measurement error. This chapter is the mechanical part: what each bias is, how to detect it, and the standard mitigation for each.

The three biases:

1. **Position bias** — in pairwise judging, the judge prefers whichever response appears first (or, for some judges, whichever appears second).
2. **Length bias** — the judge prefers longer responses regardless of quality, in both absolute and pairwise modes.
3. **Self-preference bias** — the judge scores outputs from its own model family higher than outputs from other model families, even when the outputs are equivalent quality.

All three are documented in the primary literature (Zheng et al. 2023 for position and length; Panickssery et al. 2024 for self-preference), and all three have low-cost standard mitigations that belong in any serious judging pipeline.

## Position bias and swap-and-average

The effect: for the same pair `(A, B)`, judges frequently return different verdicts depending on which response is shown first. Zheng et al. 2023 measured this on MT-Bench pairwise mode with GPT-4 and reported flip rates in the 15–40% range depending on the task category — a systematic effect large enough that ranking two comparably-good systems is essentially noise if you only judge each pair once.

The mitigation is *swap-and-average*: judge each pair twice, once as `(A, B)` and once as `(B, A)`, and aggregate the two verdicts.

The mechanics for a ternary verdict space:

```python
from collections import Counter

def swap_and_average(judge, prompt, resp_a, resp_b):
    v1 = judge(prompt, resp_a, resp_b)   # A shown first
    v2 = judge(prompt, resp_b, resp_a)   # B shown first — note the swap

    # Normalize v2 back to the original A/B labels:
    v2_normalised = {"A": "B", "B": "A", "tie": "tie", None: None}[v2]

    if v1 == v2_normalised:
        return v1, "consistent"
    if v1 is None or v2_normalised is None:
        return None, "parse_failure"
    return "tie", "swap_inconsistent"   # the pair is order-sensitive
```

Two important design decisions in that snippet.

First, the *inconsistency case* — the judge said A first time and B second time — is collapsed to `tie`. This is the correct move: the judge does not have a stable preference on this pair, so the pair carries no ranking signal. Alternatives (majority-vote across more than 2 orders, weighted averaging) exist but the tie collapse is the standard and the one Zheng et al. 2023 use in their MT-Bench pairwise numbers.

Second, the *inconsistency rate itself* is a first-class metric. If 25% of your pairs are `swap_inconsistent`, you have a rubric problem or a judge-capability problem — the judge cannot stably distinguish these pairs. Report the rate on every pairwise eval. A rate over ~30% on close comparisons is normal on hard tasks; a rate over 40% suggests the criteria are too subjective for the judge to apply reliably.

For graded verdicts (A>>B, A>B, tie, B>A, B>>B), the swap logic is the same with a symmetric relabel. Consistency requires both order-agreement and intensity-agreement; average intensity across orders when they agree.

## Length bias and length-normalised scoring

The effect: judges (both frontier and small) tend to prefer longer responses. On MT-Bench, Zheng et al. 2023 report that a padded, more verbose response often wins over a shorter one even when the two are functionally equivalent; the effect replicates on internal rubrics and is one of the most persistent judge biases.

There are two independent mitigations, and you generally want both.

**Rubric-side: the anti-length line.** Add an explicit clause to the rubric prompt: "Do not prefer a longer response merely because it is longer; prefer the one that better addresses the criteria above." Cheap. Reduces length bias measurably but does not eliminate it.

**Aggregation-side: length-normalised scoring.** For absolute rubrics, compute the correlation between response length and judge score across your eval set. If the correlation is high (Pearson |r| > ~0.3), you have a length-bias problem. The standard fixes:

1. **Bucket by length and report per-bucket scores.** Split responses into length quartiles and report the judge score per quartile. If the score is monotonically increasing with length even on items where a shorter response is *known* to be equally good, the judge is length-biased.
2. **Regress out length.** Fit `score ~ length` on the eval set and report residuals as the length-adjusted score. The residual is what the judge thinks of a response *controlling for its length*. This is the same technique used to length-adjust win rates in reward-model literature (e.g., Dubois et al. 2024, "Length-Controlled AlpacaEval").
3. **Report both raw and length-controlled numbers.** Publish the length-adjusted score alongside the raw score. The gap between them is the measurement of how much length is doing the work.

For pairwise rubrics, the analogous check is: compute the fraction of wins that go to the longer response, and compare to 50%. On a well-calibrated judge over a balanced eval set, this fraction should be near 50% (or wherever the *true* longer-is-better rate is). If it is 65%, the judge is picking length. AlpacaEval 2's length-controlled win rate (LC-WR) is the reference operationalization; the AlpacaEval leaderboard reports both raw win rate and LC-WR, and the two often disagree materially.

Pairwise length control is harder than absolute length control because the "correct" longer-is-better rate is unknown a priori. The pragmatic move is to build a small subset where you know the correct answer is length-independent (both responses correct, or both wrong, differing only in verbosity) and check the judge's win rate on that subset. If it is close to 50%, the judge is not length-biased on your task; if it is 70%, it is.

## Self-preference bias

The effect: a judge scores outputs from its own model family higher than outputs from other model families, even when the outputs are equivalent quality. Panickssery et al. 2024 ("LLM Evaluators Recognize and Favor Their Own Generations") is the primary reference. They measured this across GPT-4, Claude, and Llama-family judges scoring outputs from the same and other families, and found a consistent self-preference of roughly 3–10 percentage points in win rate depending on the task and judge, correlating with the judge's ability to *recognize* its own outputs.

Two consequences for a judging pipeline:

**Never use the same model as both subject and judge.** This is the loud one. If your subject model is `gpt-4o` and your judge is also `gpt-4o`, the judge's verdict is inflated. Use a different judge (a different family, ideally) whenever the subject appears in your candidate pool. On a pairwise leaderboard where multiple candidate models appear, this means the judge should be from a family that is *not among the candidates*.

**Run a self-preference diagnostic on your judge.** Even when subject and judge differ, if the judge and subject are from the same family (e.g., subject `gpt-4o`, judge `gpt-4-turbo`), you may still be inheriting some self-preference. The standard diagnostic is:

1. Take a small held-out set of prompts (30–100).
2. Generate one response from each candidate model (or model family) per prompt.
3. Run the judge on all pairs, blinded to model identity.
4. Compute per-family win rate.
5. Repeat with a different judge, ideally from a different family.
6. If the two judges disagree on the family ranking in a way that correlates with judge family, self-preference is in play.

The gold-standard version of this is a *panel-of-judges* setup — three or more judges from different families, aggregated by majority vote. Panel judging is expensive but eliminates single-judge family bias by construction, and is the standard when the stakes justify it (release-gating evals, safety-relevant evals).

## Putting the three controls together

A minimum-viable bias-controlled pairwise judging pipeline:

```python
def judge_pair_controlled(judge, prompt, resp_a, resp_b):
    """
    Runs a pairwise judgment with position-bias control and metadata for
    downstream length-bias analysis. Assumes judge and subject models are
    from different families (self-preference control is a pipeline-level
    concern, not a per-call one).
    """
    v1 = judge(prompt, resp_a, resp_b)                 # A first
    v2_flipped = judge(prompt, resp_b, resp_a)         # B first
    v2 = {"A": "B", "B": "A", "tie": "tie", None: None}[v2_flipped]

    if v1 == v2:
        verdict = v1
        consistent = True
    else:
        verdict = "tie"
        consistent = False

    return {
        "verdict": verdict,
        "consistent": consistent,
        "len_a": len(resp_a),
        "len_b": len(resp_b),
        "len_ratio": len(resp_a) / max(len(resp_b), 1),
        "raw_v1": v1,
        "raw_v2": v2,
    }
```

Aggregation at the eval-set level then does two things:

- Reports the mean win rate for A over B (or per-system rating in a multi-system arena, Chapter 6).
- Reports the *swap-inconsistency rate*, the *length-conditional win rate*, and the *per-judge family win rate* (if a panel is in use) as first-class outputs of the eval, not footnotes.

An eval report that shows `A wins 62% of the time` without any of those three companion numbers is unfinished. An eval report that shows `A wins 62% of the time, swap-inconsistency 12%, length-controlled win rate 58%, agreement between two judges from different families 91%` is a defensible number.

## What these controls do not fix

They control for *these three biases*. They do not fix:

- **Rubric ambiguity** — if the rubric is genuinely underspecified, position controls and length controls will not save it. Fix the rubric (Chapter 2, Chapter 3).
- **Reference-quality issues** — if the reference answer used in the rubric is wrong, the judge will confidently score against the wrong target. This is a data problem, not a bias-control problem.
- **Distribution mismatch** — if your judge has been well-calibrated on general chat data but you are grading, say, code correctness on domain-specific programs, the judge's baseline agreement with humans may be low regardless of how well you control the three biases. This is the calibration question, and it is what Chapter 5 is about.
- **Novel biases** — a judge trained after this chapter was written may have new systematic biases (formality bias, first-person bias, refusal-preference bias) not on the list above. The moral: keep your calibration slice fresh, and treat any large delta between judge score and human score on a specific slice as a hypothesis about a new bias worth investigating.

## Summary

Judge models have three well-measured systematic biases with cheap standard controls. Position bias in pairwise judging is controlled by *swap-and-average*: judge every pair in both orders, collapse order-inconsistent pairs to `tie`, and report the inconsistency rate as a first-class metric. Length bias is controlled by explicit anti-length rubric wording and by length-conditional or length-adjusted reporting; AlpacaEval's LC-WR is the reference. Self-preference bias is controlled by never using the same model as both subject and judge, running a self-preference diagnostic across judge families, and using a panel-of-judges when stakes justify it. None of these fix rubric ambiguity, reference-quality issues, or distribution mismatch — those are the calibration question, which the next chapter opens.
