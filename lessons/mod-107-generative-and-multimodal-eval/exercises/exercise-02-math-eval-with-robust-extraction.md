# exercise-02: Math Eval with Robust Extraction

**Estimated effort:** 3 hours

## Objective

Build a math-reasoning eval whose *load-bearing part is the extractor*, not the model. You will evaluate one instruction-tuned model on GSM8K and MATH, ship a layered extraction pipeline (convention → prose → numeric fallback → normalization), quantify the per-layer contribution to accuracy, add a `sympy` CAS-equivalence layer for MATH, and score a 200-item sampled subset with a chain-of-thought rubric that separates *right answer, right reasoning* from *right answer, wrong reasoning*.

## Prerequisites

- mod-107 Chapter 3 (math reasoning: extraction and CoT rubrics).
- mod-105 Chapters 2, 5 (rubric design, judge calibration) — useful for the CoT rubric part.
- Access to at least one instruction-tuned model of reasonable capability (Llama-3-Instruct-70B or better, Mixtral 8x22B, Claude Haiku, GPT-4o-mini, DeepSeek-V3, or similar).
- Access to a *judge* model for the CoT rubric part — either a stronger model than your subject, or a dedicated grader like Prometheus 2.

## The benchmarks

Use both **GSM8K** (Cobbe et al. 2021, 1319 test items) and **MATH** (Hendrycks et al. 2021, 5000 test items). For the CoT rubric subsample, use the MATH `test` split so you can compare per-subject.

Do not use MMLU-Math, MGSM, or MathVista for this exercise. GSM8K and MATH are the two benchmarks whose extraction conventions the field has explicitly agreed on and whose reference numbers exist in dozens of published reports.

## Requirements

### Part A — the layered extractor

Ship `extractor.py` exposing:

```python
def extract_gsm8k(text: str) -> tuple[str | None, str]:
    """Return (extracted_answer, method) where method names the layer that
    matched. method is one of: "boxed", "hashhash", "answer_is", "final_line",
    "last_number", None."""

def extract_math(text: str) -> tuple[str | None, str]:
    """Same shape; method includes "boxed", "answer_is", "sympy_parse", None."""
```

Each extractor is a layered pipeline that tries patterns in order and stops at the first match. At minimum:

**GSM8K extractor layers:**

1. `#### N` — the reference convention.
2. `\boxed{N}` — the model may follow the MATH convention instead.
3. `answer is [:\s]*(-?[0-9,]+(?:\.[0-9]+)?)` — prose form (case-insensitive).
4. `final answer[:\s]*(-?[0-9,]+(?:\.[0-9]+)?)` — alternate prose form.
5. Last numeric token in the last non-empty line.
6. Return `(None, None)` if nothing matches.

**MATH extractor layers:**

1. `\boxed{...}` — the reference convention (contents can be arbitrary LaTeX).
2. `answer is [:\s]*(...)` — prose form; capture until sentence end.
3. `final answer[:\s]*(...)` — alternate prose form.
4. Last mathematical expression in the last non-empty line.
5. Return `(None, None)` if nothing matches.

Then a **normalizer** for each family:

- `normalize_gsm8k(s)`: strip commas from digits, strip trailing units (`apples`, `kg`, `%`), strip whitespace, coerce to int if possible, else float.
- `normalize_math(s)`: strip LaTeX whitespace directives (`\left`, `\right`, `\!`, `\,`), canonicalize `\dfrac` / `\tfrac` → `\frac`, canonicalize `\sqrt`, remove redundant braces, normalize sign order.

Ship `test_extractor.py` that hits every layer with well-formed inputs and adversarial inputs (prose with intermediate numbers, `\boxed{}` inside a wider expression, trailing markdown, spelled-out numbers). The parser must return `None` on pathological output — a `None` return is *correct behavior* on unparseable output; guessing is not.

### Part B — the `sympy` CAS equivalence layer

Ship `math_equivalence.py` with:

```python
def is_equivalent(pred: str, gold: str, timeout_s: float = 3.0) -> bool:
    """Return True if pred and gold are mathematically equal (not just string
    equal). Uses sympy.sympify / parse_latex under a wall-clock timeout."""
```

The function attempts to parse both `pred` and `gold` into `sympy` expressions and checks `sympy.simplify(a - b) == 0` (or `set(a) == set(b)` for expressions with multiple solutions). Guard with a hard wall-clock timeout — `sympy.simplify` can hang on adversarial input. On timeout, fall back to string equality on the normalized form.

Include unit tests covering:

- `(x-1)(x+1)` vs `x^2 - 1` → True.
- `\frac{1}{2}` vs `0.5` → True.
- `\sqrt{4}` vs `2` → True.
- `\pi` vs `pi` (assuming your normalizer aligned them) → True.
- Unequal expressions → False.
- A pathological expression that would hang `sympy.simplify` → returns False (via timeout), not a hang.

### Part C — the run driver

Ship `run.py` that:

1. Loads GSM8K and MATH from their reference distributions.
2. For each item, prompts the subject model with a system message instructing "put your final answer in `\boxed{}`" and the problem. Use greedy decoding (temperature = 0) for the headline number.
3. Extracts the answer via the layered pipeline. Records the layer that matched.
4. Scores:
   - **GSM8K:** integer equality after normalization.
   - **MATH:** try string equality on normalized form first; if that fails, try `is_equivalent` (CAS). Record which of the two ruled.
5. Aggregates and reports:
   - Overall accuracy.
   - Accuracy over samples where the extractor matched the primary convention only.
   - Format compliance rate (fraction where the primary convention matched).
   - Per-layer contribution: what fraction of *correct* answers came from each extractor layer, and what fraction of *incorrect* answers had `None` extraction.
   - For MATH: the string-normalization ceiling vs. the CAS ceiling.

### Part D — the CoT rubric subsample

Sample 200 MATH items uniformly across the seven subjects (Algebra, Counting & Probability, Geometry, Intermediate Algebra, Number Theory, Prealgebra, Precalculus). For each sampled item, run the CoT rubric via a judge model.

Ship `rubric/cot_rubric.md` — the rubric prompt template, following mod-105 Chapter 2:

- Four criteria, one score each: **setup** (correct problem interpretation), **method** (appropriate technique), **arithmetic** (numerically correct calculation), **final** (correctly stated final answer).
- Anchored scoring (0/1 per criterion, or 0/1/2 if you need a partial-credit level).
- CoT-before-verdict, bounded parseable output.

Ship `rubric/run_rubric.py` that runs the judge over the 200 items and records per-criterion labels.

Report:

- Per-criterion accuracy across the subsample.
- Joint accuracy ("all four criteria correct").
- The two most interesting patterns: (a) right answer + wrong method rate (probable coincidence solves), (b) right method + wrong arithmetic rate (should-have-been-right cases).

### Part E — the report

Ship `REPORT.md` (≤ 3 pages) covering:

1. **Setup.** Subject model, judge model, decoding params, dataset versions.
2. **Headline numbers.** GSM8K accuracy, MATH accuracy (string-normalization and CAS), format compliance rate.
3. **Extractor contribution table.** Rows = extractor layers, columns = (fraction of samples matched, accuracy on samples matched). Comment on which layers are load-bearing.
4. **Per-subject MATH accuracy.** A row per subject.
5. **CoT rubric findings.** Per-criterion accuracy, joint accuracy, the two rate patterns from Part D. Include 3–5 example rubric-scored items in an appendix that make the patterns concrete.
6. **What breaks the extractor.** 3–5 sample outputs where the extractor missed something a human would extract, with a proposed fix for each.
7. **Reproducibility manifest.** Everything.

### Part F — bundle

- `extractor.py`, `test_extractor.py`
- `math_equivalence.py`, `test_math_equivalence.py`
- `run.py`
- `rubric/cot_rubric.md`, `rubric/run_rubric.py`, `rubric/logs.jsonl`
- `logs/gsm8k.jsonl`, `logs/math.jsonl`
- `REPORT.md`
- `README.md` — rerun instructions

## Starter guidance

- **Write the extractor before running the benchmark.** Build `extractor.py` and `test_extractor.py` against a set of hand-authored fake outputs (10–20 that hit each layer, plus adversarial inputs). Only then run the benchmark. Otherwise you will retrofit the extractor to explain whatever the model happens to produce.
- **Extraction failures are not model failures.** The metric you care about first is *format compliance*. If format compliance is 60%, the model is right more often than the overall-accuracy number suggests; the fix is either a stronger format instruction or a more permissive extractor. Do not conflate the two.
- **Report the ceiling gap on MATH.** String-normalization accuracy vs. CAS-equivalence accuracy is a 1–5 point gap on most modern models. That gap is the extractor's headroom.
- **Do not trust `sympy.parse_latex` blindly.** The parser is fragile on adversarial LaTeX (nested `\left`/`\right`, malformed environments, competition-math shorthand). Guard with a timeout, catch exceptions, and log parse failures. `is_equivalent` returning False on a pathological parse is fine; hanging the whole run on it is not.
- **Use a *different* judge model than your subject for the rubric.** Self-preference (mod-105 Chapter 4) contaminates the rubric numbers if judge and subject share a family. Ideally use a frontier model (a tier above your subject) or a dedicated grader like Prometheus 2.
- **Anchor the rubric carefully.** The most common rubric-design mistake for CoT scoring is conflating "arithmetic error" and "wrong method." Write out explicit examples for each criterion in the anchor text; test the rubric on 10 hand-authored items before running the full 200.
- **Format compliance moves with the prompt.** "Put your final answer in `\boxed{}`" vs. "End with a line reading 'ANSWER: <value>'" vs. no instruction at all produce very different compliance rates. Log the prompt template alongside the compliance number.
- **A 5-point gap between extracted-accuracy and overall-accuracy is a bug.** Either loosen the extractor or tighten the format. Neither is model-quality; both are eval-quality.

## Acceptance criteria

- `extractor.py` handles every layer for both benchmarks and returns `None` on pathological output (verified in `test_extractor.py` with at least 2 adversarial cases per benchmark).
- `math_equivalence.py` handles the six CAS cases in Part B and does not hang on adversarial input.
- `run.py` reports overall accuracy, format compliance, per-layer contribution, and the string-vs-CAS ceiling gap on MATH.
- The 200-item CoT rubric subsample is scored per-criterion with joint accuracy and the two rate patterns from Part D.
- `REPORT.md` includes the extractor contribution table, per-subject MATH accuracy, and the CoT rubric findings with example items.
- The bundle reproduces: a reviewer can rerun `run.py` and `rubric/run_rubric.py` and get numbers within decoding-noise (usually ~1 point on greedy).

## Stretch goals

- **Self-consistency `acc@maj-N`.** Repeat the GSM8K run at temperature 0.6 with `N = 32` samples per problem, majority-vote extracted answers. Report `acc@greedy` vs. `acc@maj-32`. The lift is often 3–7 points on smaller models, less on frontier ones.
- **A second subject model.** Rerun everything with a second model of comparable class. Compare per-subject profiles on MATH — the two models often differ by 5–10 points on specific subjects, which is exactly the kind of profile you cannot see from the aggregate.
- **GSM1k (Zhang et al. 2024) alongside GSM8K.** GSM1k is a fresh matched-difficulty benchmark specifically for measuring contamination. If accessible, run it and report the GSM8K → GSM1k gap. A gap > 10 points is a strong contamination signal.
- **Judge calibration slice.** Human-label 50 items yourself on the CoT rubric. Compute per-criterion κ between you and the judge. Report κ alongside the rubric numbers. Below κ = 0.6, treat the rubric numbers as diagnostic-only, not gate-quality.
- **MATH subject weighting.** Recompute the MATH aggregate as a weighted mean using per-subject weights that match what your product surface actually needs (e.g. 40% algebra, 30% geometry, 10% each of the remaining three, if you're building a K-12 tutor). The weighted number is more meaningful than the uniform mean; the writeup should say so.
