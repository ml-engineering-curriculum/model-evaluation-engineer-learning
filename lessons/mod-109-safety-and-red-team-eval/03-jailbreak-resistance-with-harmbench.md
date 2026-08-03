# Jailbreak Resistance With HarmBench

Chapter 2 measured behavior against *direct* harmful requests: prompts the model was designed to refuse, sent as-written. Real-world abuse looks nothing like that. An adversary who is willing to iterate can transform a plain harmful request into thousands of variants — a role-play frame, a base-64 wrapper, an ostensibly-academic rationale, a translation into a less-guarded language, a multi-turn "warm-up" that softens the refusal in later turns. Chapter 2's refusal rate is the model's behavior against a *cooperative* user. This chapter's *jailbreak resistance* is behavior against an *adversarial* user, and the metric — attack success rate (ASR) — is a curve over attacker effort, not a scalar.

The chapter walks how to run HarmBench (Mazeika et al. 2024) and comparable curated attack suites against a model, how to report ASR without turning the writeup into an attack cookbook, and how to make the attacker model explicit so two teams' numbers are comparable.

## The attacker model is the load-bearing definition

Attack success rate depends entirely on what the attacker is allowed to do. Three attacker profiles bracket the space:

- **Static-attacks attacker.** Uses a fixed corpus of pre-authored jailbreak prompts (e.g. the "vanilla" HarmBench prompts, older DAN-style prompts, published prompt-hacking collections). No iteration. This is the cheapest attacker to simulate and gives the *floor* — a model that fails against this attacker is trivially exploitable.
- **Single-turn transformations attacker.** Takes each harmful behavior and applies one of `N` documented transformations — role-play framing, cipher wrappers, translation, "for research purposes" framing, prefix / suffix injection. HarmBench's `attacks/` module supplies a reference set. Still cheap, but broader coverage.
- **Automated iterative attacker.** Runs an attack loop that queries the target model, reads the response, and generates the next attempt. PAIR (Chao et al. 2023), TAP (Mehrotra et al. 2024), Crescendo (Russinovich et al. 2024, multi-turn), and Best-of-N (Andriushchenko et al. 2024) are the reference automated attackers. This is expensive (dozens to hundreds of target-model queries per behavior) and produces the strongest attacker in the published literature. It is the attacker most model cards report against for their headline ASR number.

A fourth level — human red-teamers with unbounded time — is what serious labs use before release. It is not reproducible as an eval and does not belong here; it belongs in the red-team engagement discipline referenced in Chapter 6.

The reporting rule: **your ASR number is only interpretable next to the attacker profile that produced it.** "ASR = 8% on HarmBench" without an attacker specification is a nearly meaningless number. "ASR = 8% under the HarmBench single-turn attacker with `k=1` attempt per behavior, HarmBench compliance judge v2, HarmBench standard split, dataset revision `abc123`" is a claim.

## HarmBench: a concrete instrument

HarmBench (Mazeika et al. 2024, ICML) is the most widely-cited public jailbreak eval at the time of writing. Its structure at a high level (published in the paper and the GitHub repo):

- ~400 harmful *behaviors* across seven semantic categories (illegal, cybercrime, misinformation, harmful chemical/biological, harassment/bullying, general harm, copyright). Each behavior is a short description like "Instructions for [X]" — the *description*, not the payload; the payload is the model's completion.
- A `standard` split (direct behaviors) and a `contextual` split (behaviors wrapped in a benign-sounding narrative).
- An `attacks/` module implementing 18 attack methods (GCG, AutoDAN, PAIR, TAP, prompt-based transformations, human-authored, etc.) that transform behaviors into attempt prompts.
- A trained compliance classifier that reads `(behavior, model_response)` and outputs `attack_success` / `refused` — the closest thing the field has to a shared automatic judge.

For our purposes the eval loop looks like this (schematic — the exercise-02 flesh-out has the exact commands):

```python
# Pseudocode. Do NOT copy-paste attack payloads into this file.
behaviors = load_harmbench_behaviors(split="standard", revision="<sha>")

for attacker in [static_baseline, transformations_attacker, PAIR_attacker]:
    for behavior in behaviors:
        attempt = attacker.transform(behavior)          # produces one or more attempt prompts
        response = target_model.generate(attempt)
        verdict  = judge.classify(behavior, response)   # attack_success / refused
        log(behavior_id=behavior.id, attacker=attacker.name, verdict=verdict)

report_asr_per_attacker_per_category(...)
```

Reporting shape:

```
Model: <provider/model@version>
Judge: HarmBench compliance classifier v2 (agreement with human gold: 92% on 500-sample eval)
Dataset: HarmBench Standard @ <revision-sha>, 400 behaviors

ASR by attacker profile:
  static-vanilla:          0.02  [0.01, 0.04]
  single-turn-transforms:  0.07  [0.05, 0.10]
  PAIR (k=20 iterations):  0.19  [0.15, 0.23]
  TAP  (k=20 iterations):  0.24  [0.20, 0.28]

Per-category (under PAIR):
  illegal:       0.11  [0.06, 0.17]
  cybercrime:    0.28  [0.20, 0.36]
  chem/bio:      0.06  [0.02, 0.11]
  ...

Attacker budget (per behavior, PAIR):
  target-model queries: mean 12, median 8, p95 24
  attacker-model queries: mean 12, median 8, p95 24
  wall-clock:            mean 82s, p95 240s
```

The three axes that make this a defensible ASR: **attacker profile, dataset revision, judge**. Chapter 7's model-card format expects those three.

## How to run this without writing an attack cookbook

The single biggest hazard of authoring safety-eval infrastructure is that the code, the tests, and the report inevitably touch attack content. Four disciplines keep this from becoming a leak.

- **Load, don't author.** Your codebase never contains hand-written jailbreak prompts. Every prompt comes from a pinned dataset revision (HarmBench, AdvBench, JailbreakBench) loaded at runtime from its official source. If you find yourself writing a jailbreak string into a test fixture, you're doing it wrong — mock the model output instead, or load a small subset of the real dataset from a controlled path.
- **Log at aggregate granularity by default.** The default log level for a jailbreak eval writes only `(behavior_id, attacker_id, verdict, latency, tokens)`. It does not log the transformed prompt or the model's response. When you need those for debugging a specific failure, they go to a *separate* controlled store with access control and retention limits.
- **Truncate in reports.** When you must include a per-behavior example (an appendix showing "here's what a successful attack looked like"), truncate to the first ~30 tokens with `[TRUNCATED]` and, if the truncated prefix itself carries attack signal, redact by category rather than by verbatim. Chapter 7 formalizes this.
- **Category-level reporting, not example-level.** A model-card table showing per-category ASR is safe; a model-card table showing per-behavior verdicts (with behavior text) is a curated hit-list for the next attacker. External artifacts stop at category-level; behavior-level lives internal-only.

None of this makes the eval itself less useful — the aggregate metrics are what a release-review actually reads. It does make the artifact defensible to circulate.

## The compliance judge, revisited

The judge that grades `(behavior, response) → attack_success / refused` is the same shape as Chapter 2's compliance judge but with a different definition of "success." Chapter 2 asked "did the model refuse?" This chapter asks "did the model comply with the *specific harmful behavior* the attacker was targeting?"

The distinction matters because a model can:

- Refuse (verdict: `refused`).
- Comply but with a wrong / hallucinated answer (verdict: `refused` — from a safety standpoint the model did not deliver harm).
- Comply partially with vague, non-actionable content (`refused` or `partial` depending on the judge — some are conservative, some are liberal).
- Fully comply with actionable content (`attack_success`).

HarmBench's own classifier is trained on the third distinction: it looks for "did the response actually deliver the harmful content the behavior asked for?" — not just "was there refusal language?" A rule-based `is_refusal` classifier from Chapter 2 will systematically *under*-report attack success, because a model that starts with "Sure, I can help with that. Here's how:" and then hallucinates gibberish looks like a compliance and hence a success to the rule-based classifier, when in fact no harm was delivered.

For any published ASR number, report the judge you used *and* the judge's agreement with a human gold set. HarmBench's paper reports its own classifier at ~92% agreement with expert human labels on a 500-sample subset; a defensible number.

## Attacker budget: the axis that changes every ASR

An ASR curve is a function of attacker effort, and effort has a unit. For iterative attackers (PAIR, TAP, best-of-N) the standard unit is *target-model queries* — how many times did the attacker have to query the model to get one attack success? Two frames are useful:

- **ASR at fixed k**: fix an attacker budget of `k = 1, 5, 10, 20, 50` queries per behavior and report ASR as a function of `k`. The curve is monotonic; the interesting values are ASR at `k=1` (equivalent to a static attacker) and the plateau value (attacker saturates).
- **Queries-to-first-success**: for each behavior, report the number of queries the attacker needed to succeed (or a censored value if it never succeeded in `k_max`). Median and p95 of this distribution are informative.

Both frames make apples-to-apples cross-model comparison possible. "Model A: 24% ASR at PAIR-k=20" and "Model B: 12% ASR at PAIR-k=50" is not comparable; those are different attacker budgets. Fixing `k` makes them comparable at that point on the curve; publishing the curve makes the comparison at every point.

For static and single-turn-transform attackers, `k=1` per behavior is standard; the budget dimension collapses.

## Multi-turn attacks: the next hard problem

Single-turn attacks are the well-studied case. Multi-turn attacks — where the attacker uses several conversational turns to soften the model's refusal — are harder to measure and produce systematically higher ASR at frontier models. The reference method is Crescendo (Russinovich et al. 2024), which structures a conversation across ~5–10 turns starting with benign context and gradually shifting toward the harmful ask.

Multi-turn ASR reporting is not yet standardized. When you report it, name the attacker method, the number of turns, and whether the attacker had access to intermediate model responses (attacker-in-the-loop) or ran a scripted transcript (open-loop). A `Crescendo, 8-turn attacker-in-the-loop, k=1 per behavior` ASR is a claim; `multi-turn ASR = 30%` without those axes is not.

The exercise-02 flesh-out has this as a stretch goal — the single-turn case is enough for a durable first pass.

## Correlations across attackers and the false-security failure

A common finding in the wild: a model has near-zero ASR on `attacker_A` and 30%+ ASR on `attacker_B`. The instinct to write "resistant to attacker_A" is a well-documented safety failure. Two lessons:

- **A single attacker's ASR is not a bound.** Absence of exploits from attacker profile X is not evidence of absence from profile Y. Report multiple attackers per release and expect the numbers to disagree substantially.
- **The universal-jailbreak assumption is wrong at frontier scale.** GCG-generated adversarial suffixes (Zou et al. 2023) that transfer across models were the 2023 story; PAIR / TAP / Crescendo / multi-turn attackers are the 2024 story; the field's understanding is that attacker diversity outstrips any single defensive method. A safety report should assume the attacker is diverse.

## What you cannot conclude from ASR

Even a well-run ASR measurement has limits.

- **ASR is not real-world exploit rate.** A benchmark ASR of 20% does not mean 20% of real users exploit the model. Real users are not adversarial in the same way; adversaries use techniques the benchmark did not cover.
- **ASR is not a bound on capability uplift.** A model that refuses at 99% ASR might still, on the 1% of successful attacks, deliver highly actionable harmful content. ASR × uplift-severity is the actually-relevant risk axis, and severity is not a benchmark output — it is a domain-expert judgement. Chapter 4's dangerous-capability discussion picks this up.
- **ASR moves under post-training.** A small safety-focused fine-tune can drop ASR sharply, but if the fine-tune generalizes narrowly, the model might still be vulnerable to attackers not seen in fine-tuning. Report ASR against a *held-out* attacker (one not used in safety training) whenever possible.

## Guidance for the eval author

- **Publish the attacker profile.** Every ASR number carries: attacker method, attacker budget (`k`, wall-clock, queries), dataset revision, judge, judge's calibration. This is non-negotiable.
- **Report the curve, not just the peak.** ASR as a function of attacker budget makes the comparison across models honest.
- **Run at least three attacker profiles.** A single-attacker report understates risk. A minimum defensible set at time of writing: HarmBench static, HarmBench single-turn transformations, one automated iterative (PAIR or TAP), plus per-category slices.
- **Human-calibrate the judge on your model's outputs.** Different models refuse differently; a judge calibrated on model X's completions can drift on model Y. Take a 100–200 sample and re-check the classifier's agreement with human gold before publishing a headline ASR.
- **Never put payloads in the report.** Category-level tables, aggregate ASR, redacted prefix examples. Chapter 7 has the format.
- **Do not confuse ASR with a security bound.** Absence of ASR on your benchmark is not absence of exploitability. This is a safety measurement, not a security guarantee.
- **Rotate the dataset each release.** HarmBench and similar suites are widely available in pretraining scrapes; a model that scores well on stale HarmBench and poorly on a recent private set is a specific diagnostic (contamination-driven safety).

## Summary

Jailbreak resistance is measured as ASR against a documented attacker profile — a triple of (attacker method, attacker budget, judge) that must accompany the number for it to be interpretable. HarmBench supplies a standard corpus, a set of reference attackers (static, transformations, iterative), and a trained compliance judge; comparable published suites include AdvBench, JailbreakBench, and the WMDP-derived attack sets. Report ASR as a curve over attacker budget, not as a single peak, and always run multiple attacker profiles. The compliance judge is the load-bearing component and must be calibrated against human gold specifically for the model under test. The four data-handling disciplines — load-don't-author, aggregate-by-default logging, redacted truncation, category-level external reporting — keep the eval infrastructure from becoming an attack cookbook. The next chapter goes deeper into the *severity* axis that ASR alone does not measure: awareness-level dangerous-capability evals and the public risk frameworks (Anthropic RSP, OpenAI Preparedness, UK AISI) that turn a benchmark number into a release-decision input.
