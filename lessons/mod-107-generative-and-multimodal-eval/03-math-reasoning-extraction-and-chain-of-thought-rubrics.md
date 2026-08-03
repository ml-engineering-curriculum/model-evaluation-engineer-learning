# Math Reasoning: Extraction and Chain-of-Thought Rubrics

Math benchmarks look the friendliest of the four families in this module. GSM8K (Cobbe et al. 2021) gives you a grade-school word problem and a single canonical integer answer. MATH (Hendrycks et al. 2021) gives you a competition-style problem and a canonical expression (often a fraction, a surd, or a small algebraic form). The metric is answer accuracy. There is no test suite to run, no retriever pipeline to instrument, no image to ground against. It looks like the simplest thing in this module.

It is not. The chapter's argument is that on math benchmarks, the *answer extractor* is doing more work than the model. A modern model does not emit "42" — it emits several paragraphs of chain-of-thought ending in something like `Therefore the answer is $\boxed{42}$.` or `So the answer is 42 apples.` or `The final answer is: 42.` — and any of those can be missed by a naive extractor. The published-vs-reproduced gap on GSM8K and MATH is often 5–15 points wide, and most of that gap is extraction, not reasoning. The chapter walks the extractor first, then the second layer — *chain-of-thought rubrics* that score the process rather than just the final answer — and closes with the practical decision of when each is worth its cost.

## The two canonical benchmarks

**GSM8K** (Cobbe et al. 2021) is 8,500 grade-school word problems (~7.5k train, 1,319 test) with integer answers. The reference format ends every solution with `#### 42` — a delimiter designed to be trivially extractable. The benchmark's default eval script splits on `####` and compares integers.

**MATH** (Hendrycks et al. 2021) is 12,500 competition-math problems (7.5k train, 5k test) drawn from AMC / AIME / high-school Olympiad sources. Solutions are formatted in LaTeX and the canonical answer is wrapped in `\boxed{}`. The reference evaluator strips `\boxed{...}`, normalizes whitespace and LaTeX conventions (spaces around operators, `\dfrac` vs `\frac`, `\pi` vs `pi`), and compares the normalized strings.

Both benchmarks were designed with an *extraction convention* baked into the reference solution: `#### N` for GSM8K, `\boxed{...}` for MATH. Modern instruction-tuned models frequently ignore both. A model told "solve this problem" without a format instruction will finish with prose. This is the extraction problem in one sentence: the benchmarks agreed on a convention, and the models the benchmarks now score do not follow it by default.

## The extraction problem

Consider these five plausible endings a model might produce for the GSM8K problem *"Alice has 12 apples..."*:

1. `#### 15` — matches the GSM8K convention exactly.
2. `The answer is 15.` — needs prose extraction.
3. `Therefore Alice has $\boxed{15}$ apples.` — LaTeX box in a prose sentence.
4. `Final answer: **15 apples**.` — markdown emphasis, unit trailer.
5. `Alice ends up with fifteen apples in total.` — spelled-out number.

A naive extractor built for reference GSM8K format (`re.search(r"####\s*(-?\d+)", text)`) matches only case 1. Extending to prose (`re.search(r"answer is[:\s]*(-?\d+)", text, re.IGNORECASE)`) covers cases 1 and 2. Handling `\boxed{}` (`re.search(r"\\boxed\{(-?\d+)\}", text)`) plus prose plus `####` covers 1–3. Case 4 needs stripping of markdown and unit words. Case 5 needs number-word conversion (`fifteen` → `15`).

Each layer picks up 1–3 percentage points on modern models. On a model that consistently produces case 3, a case-1-only extractor scores it at close to zero. This is not a hypothetical: early GSM8K reports from teams that used the official Cobbe et al. eval script on instruction-tuned models saw suspiciously low scores until the community converged on the pattern of adding a `\boxed{}` prompt instruction *plus* a robust extractor.

### A layered extraction pipeline

The pattern most modern harnesses converged on:

1. **Prompt for a format.** Include an instruction like "Please put your final answer in `\boxed{}`" or "End your answer with `#### <number>`." This raises the fraction of samples that follow the convention from ~40–60% to 90–95% on cooperative instruction-tuned models. It is not enough on its own; some samples still ignore it. But it is the cheapest 10-point accuracy boost you can get.
2. **Try the convention first.** Look for `\boxed{...}` (MATH-style) or `#### N` (GSM8K-style) in the last 200 characters. This handles the majority of well-formed outputs.
3. **Fall back to prose patterns.** "The answer is X," "Final answer: X," "= X" at end-of-line, "So X" as the last non-empty line. These are ordered from most to least specific.
4. **Fall back to *last number in the last line*.** Extract the last numeric token in the last non-empty line. This is aggressive but produces correct answers on many free-form outputs; it also introduces false positives when the model reasons through several intermediate numbers. Measure the false-positive rate before shipping it as default.
5. **Normalize before comparing.** For GSM8K: strip commas from digits (`1,000` → `1000`), strip trailing units (`15 apples` → `15`), coerce to integer. For MATH: strip whitespace, canonicalize LaTeX (\dfrac → \frac, \tfrac → \frac, remove `\left`/`\right`, alphabetize `+` chains, canonicalize `\sqrt`), coerce fractions to lowest terms.
6. **Return `None` (not a wrong answer) on parse failure.** An extractor that "guesses" on ambiguous output silently inflates the wrong-answer rate. Log parse failures as a separate metric.

Every mature math-eval harness ships some version of this. lm-evaluation-harness's `gsm8k` task uses a regex chain and a fallback. The Hendrycks MATH reference eval (`math_equivalence.py`) is 200+ lines of LaTeX normalization. Minerva (Lewkowycz et al. 2022) contributed an improved MATH normalizer that the community folded in. The extractor is a real artifact; treat it as such.

### The `sympy` route for MATH

For MATH specifically, string normalization has a ceiling: two expressions that are string-different but *mathematically* equal (e.g. `(x-1)(x+1)` vs `x^2 - 1`) will fail equality. The higher-precision route is to parse both the extracted answer and the reference into a computer-algebra representation (via `sympy.sympify` or `sympy.parse_latex`) and check `sympy.simplify(a - b) == 0`. This is slower and can hang on adversarial expressions (guard with a timeout), but it produces the ceiling of a string-normalization pipeline. Publish both numbers — string-normalized accuracy and CAS-equivalence accuracy — and the gap tells you how much extraction noise remains.

## Chain-of-thought rubrics: process, not just outcome

Answer accuracy scores the outcome. Two well-known failure modes surface when outcome is all you score:

- **Right answer, wrong reasoning.** The model coincidentally lands on the correct integer via a broken derivation. On GSM8K's arithmetic this is rare; on MATH's competition problems it is not. A right-answer-wrong-reasoning sample counts as a correct answer under outcome-only scoring but is not a signal that the model can solve the problem class.
- **Wrong answer, right reasoning.** The model derives correctly and misarithmetics the last step, or misreads the question, or answers the wrong sub-question. Outcome-only scoring treats it as fully wrong; process scoring treats it as partial credit and preserves the information that the *reasoning* is fine.

Chain-of-thought rubrics score the reasoning trace itself. The rubric anchors ("correct setup," "arithmetic error in step 3," "wrong problem interpretation," "correct method wrong final step") come from mod-105 Chapter 2. The typical shape:

- **Rubric criteria.** Setup / method / arithmetic / final answer. Some rubrics collapse to a single "process quality" score; the four-criterion version is more informative and easier to calibrate.
- **Grading pass.** An LLM judge (Prometheus or a frontier judge) reads the reference solution and the model's chain of thought and returns a per-criterion label. The rubric is anchored the same way as any absolute rubric.
- **Aggregation.** Per-criterion accuracy across the benchmark, plus a *joint* number ("all four criteria correct"). Report both, because they answer different questions — per-criterion diagnoses failure modes, joint gives you a strict-correctness proxy.

Process rubrics were formalized in Uesato et al. 2022 and Lightman et al. 2023 (*Let's Verify Step by Step*), whose PRM800K dataset is the reference training set for step-level correctness labelling. The community lesson from that work was that process supervision beats outcome supervision for training reasoning models; the eval-engineering lesson is subtler — a process rubric gives you a *dashboard* per-criterion breakdown that outcome accuracy cannot, at the cost of judge inference on every sample and calibration effort against a human gold set.

Most production math evals do *not* run a full CoT rubric on every item; they run outcome-only accuracy on the full benchmark plus a sampled rubric-scored subset (200–500 items) for the diagnostic breakdown. That balance is the one recommended in Exercise 02.

## Format-following as a first-class metric

If your extractor depends on a format instruction ("put your final answer in `\boxed{}`"), then *format compliance rate* is now a metric of its own. Two systems with the same underlying reasoning ability but different format compliance rates will produce different measured accuracies — the compliant system loses fewer samples to extraction failure. That means the accuracy number confounds two things: reasoning quality and format-following quality.

Report both:

- **Format compliance rate.** Fraction of samples where the extractor matched the primary (convention) pattern.
- **Extracted accuracy.** Accuracy over the samples the extractor could parse.
- **Overall accuracy.** Accuracy over *all* samples, treating parse failures as wrong.

The gap between extracted accuracy and overall accuracy is the tax the format failures impose. If it is large, either loosen the extractor or tighten the format instruction. Do not report only one number.

## Chain-of-thought decoding: sampling and self-consistency

Sampling matters differently here than in code eval. GSM8K and MATH are typically reported at greedy or very-low-temperature decoding for the headline number — the metric is a per-problem correctness, not a pass-in-`k` diversity story. But *self-consistency* (Wang et al. 2022) — sample `n` chains of thought, extract the answer from each, majority-vote — reliably lifts accuracy on math benchmarks by several points and is standard in modern reports.

The eval-engineering implication is that you have to specify the decoding regime alongside the accuracy number:

- `pass@1 greedy` or `acc@greedy` — the model's answer under greedy decoding.
- `acc@maj-N` — majority vote of `N` samples, typically `N = 8`, `32`, or `64`.

`acc@maj-32` is often 5–10 points above greedy on GSM8K for the same model. Reporting one without the other is misleading. Modern lm-eval-harness GSM8K variants let you configure this via the task YAML; Inspect exposes it via a `Solver` composition (`self_consistency(n=32)`). Whichever harness you use, the reported number needs the decoding tag.

## Contamination is real on math

GSM8K in particular is heavily contaminated in modern pretraining corpora — the dataset is publicly available on Hugging Face and is echoed across dozens of derivative datasets. Test-set leakage on GSM8K is documented (Zhang et al. 2024, *A Careful Examination of Large Language Model Performance on Grade School Arithmetic*, introduces GSM1k as a fresh matched benchmark and shows several models drop 10+ points from GSM8K to GSM1k). MATH is somewhat cleaner because of its LaTeX-heavy format but is still subject to leakage.

The eval-engineering response, as in mod-102:

- Cite the benchmark version and any decontamination step.
- If contamination is a concern, run a matched clean benchmark (GSM1k, MATH-Perturb, fresh competition problems past the training cutoff) alongside the standard benchmark and report the gap. The gap is your contamination signal.
- Do not treat a saturated GSM8K number as a claim about math ability.

## Summary

Math benchmarks look like simple exact-match tasks, but on modern instruction-tuned models the *answer extractor* is doing most of the eval-engineering work. A robust extractor tries the benchmark's convention (`\boxed{}` for MATH, `#### N` for GSM8K) first, falls back to prose and last-number patterns, normalizes numeric and LaTeX forms, and returns `None` (not a wrong answer) on parse failure. For MATH, `sympy`-based CAS equivalence lifts the ceiling above what string normalization alone can reach. Above outcome accuracy, chain-of-thought rubrics score the *process* — setup / method / arithmetic / final — via a judge on a sampled subset, giving you a per-criterion dashboard that outcome-only accuracy cannot. Format compliance rate, extracted accuracy, and overall accuracy are three different numbers and should be reported together; the same is true for greedy accuracy versus self-consistency `acc@maj-N`. Contamination on GSM8K in particular means a saturated headline number should be validated against a clean matched benchmark before being taken as a claim about the model's math ability. The next chapter turns to RAG, where the measurement instrument stops being "answer accuracy" entirely and becomes a *pipeline-instrumentation* problem.
