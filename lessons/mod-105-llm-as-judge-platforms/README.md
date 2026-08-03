# mod-105-llm-as-judge-platforms: LLM-as-Judge Platforms: Rubrics, Bias Controls, and Calibration to Humans

**Estimated effort:** 15 hours

The previous module put you inside the four benchmark harnesses the field actually uses. Most of what those harnesses score is a log-likelihood over a fixed continuation or a normalized string match against a reference. That works for MMLU and GSM8K. It stops working the moment the eval question is "did the assistant write a good summary," "did the reply actually address the user's problem," or "which of these two answers is better for a coding task." There is no reference string to match; there is often no single right answer. LLM-as-judge platforms exist to fill that gap — use a second capable model as the scoring function, prompt it with a rubric, and treat its verdict as the metric.

The rest of the module is largely about the fact that a judge model is a *measurement instrument*, and instruments have systematic error. The chapters walk from designing rubrics whose scores mean the same thing across runs, through the bias controls that any published judge-graded number should carry (position, length, self-preference), through the calibration study that turns a plausible number into a defensible one (Cohen's κ, weighted κ, Spearman / Kendall), through the Bradley-Terry / ELO machinery behind Arena-style leaderboards, and finally to the routing decision every production stack now faces: hosted frontier, open-source general-purpose, or dedicated trained judge, and which item goes where.

## Learning objectives

- Design an absolute (single-answer) and a pairwise rubric for a real product task with explicit scoring anchors.
- Implement position-bias controls (swap-and-average), length-bias controls (length-normalised scoring), and self-preference diagnostics.
- Calibrate a judge against a human gold set (Cohen's κ, Spearman / Kendall agreement) and decide whether the judge is good enough to ship.
- Stand up an Arena-style pairwise rating system (Bradley-Terry / ELO-style) and reason about its statistical properties.
- Choose between hosted (frontier API) judges, open-source judges (Prometheus, JudgeLM), and dedicated trained judges, with cost-vs-quality routing.

## Lecture chapters

1. [`01-why-a-model-judges-a-model.md`](01-why-a-model-judges-a-model.md) — why model-graded eval exists, the absolute-vs-pairwise split, and the four objects (model adapter, task definition, request type, scorer) translated into the judge setting.
2. [`02-absolute-rubrics-with-anchors.md`](02-absolute-rubrics-with-anchors.md) — the six rules for an absolute rubric whose scores mean the same thing across judges and releases: distinct anchors, response-focused descriptions, explicit criteria, reference-in-prompt, CoT-before-verdict, and a bounded parseable output.
3. [`03-pairwise-rubrics-and-preference-framing.md`](03-pairwise-rubrics-and-preference-framing.md) — pairwise rubric design, verdict shapes (binary / ternary / graded), position-neutral labels, and the two pairwise-only failure modes (superficial tie-breaking and comparison drift on stylistically different responses).
4. [`04-bias-controls-position-length-and-self-preference.md`](04-bias-controls-position-length-and-self-preference.md) — the three systematic judge biases with their standard controls: swap-and-average for position bias, length-adjusted scoring / AlpacaEval LC-WR for length bias, and disjoint judge/subject families plus panel-of-judges for self-preference.
5. [`05-calibrating-a-judge-against-humans.md`](05-calibrating-a-judge-against-humans.md) — the anatomy of a calibration study: validation-set sampling, human-labels-first, adjudication rules, Cohen's / weighted κ vs. Spearman / Kendall, bootstrap CIs, confusion-matrix reading, and the five-check ship / rework / caveat decision.
6. [`06-arena-style-rating-with-bradley-terry-and-elo.md`](06-arena-style-rating-with-bradley-terry-and-elo.md) — turning sparse pairwise verdicts into per-system ratings: Bradley-Terry maximum-likelihood with cluster-bootstrap CIs, ELO as the online cousin, rating-proximal match scheduling, tie conventions, and the "judge accuracy is the ceiling" property.
7. [`07-choosing-and-routing-judges-hosted-oss-and-trained.md`](07-choosing-and-routing-judges-hosted-oss-and-trained.md) — the three judge tiers (hosted frontier, OSS general-purpose, dedicated trained like Prometheus / JudgeLM), the routing patterns (fixed-tier, difficulty-based escalation, panel), the cost model that includes infrastructure and human-review-of-tail, and the judge-drift discipline that every hosted-judge stack needs.

## Exercises

Five hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after the chapters it depends on.

- [`exercise-01-absolute-and-pairwise-rubric-design.md`](exercises/exercise-01-absolute-and-pairwise-rubric-design.md) — design and pilot a matched pair of rubrics (absolute + pairwise) for the same real product task, validated on a 20–30-item pilot with known-answer probes.
- [`exercise-02-position-and-length-bias-controls.md`](exercises/exercise-02-position-and-length-bias-controls.md) — measure and mitigate position, length, and self-preference bias on a real judge, with swap-and-average, length-probe subsets, and a cross-family judge comparison.
- [`exercise-03-judge-vs-human-calibration-study.md`](exercises/exercise-03-judge-vs-human-calibration-study.md) — full calibration study: 100+ items, two annotators with adjudication, κ / weighted κ / Spearman with bootstrap CIs, and an explicit ship / rework decision against the Chapter 5 five-check list.
- [`exercise-04-bradley-terry-arena-implementation.md`](exercises/exercise-04-bradley-terry-arena-implementation.md) — stand up a working arena over 5+ candidate systems: Bradley-Terry fit with cluster-bootstrap CIs, ELO comparator, rating-proximal sampler, and four experiments on the statistical properties.
- [`exercise-05-judge-tier-routing-and-cost-model.md`](exercises/exercise-05-judge-tier-routing-and-cost-model.md) — calibrate three judge tiers (hosted frontier, OSS general, dedicated trained) against the same human gold set, implement difficulty-based escalation, and produce a defensible tier-routing recommendation with a numeric cost / κ trade-off.

Reference solutions live in the paired [`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for primary references — Zheng et al. 2023 (MT-Bench and Chatbot Arena, the paper that defines the modern LLM-as-judge setup), Panickssery et al. 2024 (self-preference), Chiang et al. 2024 (Chatbot Arena methodology), Kim et al. 2024 (Prometheus 1 and 2), Dubois et al. 2024 (AlpacaEval length control), Bradley and Terry 1952 (the rating model), the OpenAI evals / lm-eval-harness / Inspect / Prometheus repos, and the sklearn / scipy documentation for the agreement statistics.
