# exercise-05: Product-Shaped Eval Suite Design

**Estimated effort:** 2 hours

## Objective

Take a stated product surface, map it to the instruments in this module and its predecessors, and produce a *suite design document* plus a minimum-viable configuration. No new eval runs are required — this exercise is entirely about the construction and writeup discipline from Chapter 7. The deliverable is a suite that a reviewer without the surrounding context can pick up, understand what it measures and *what it does not*, and rerun.

## Prerequisites

- mod-107 Chapter 7 (composing a product-shaped eval suite).
- The rest of this module (Chapters 2–6) — you're going to reach for their instruments by name.
- mod-104, mod-105, and mod-106 exist as source material for non-generative axes (classical benchmarks, judge-graded rubrics, human eval subsets).

## The product surface: pick one

Pick a real or plausible product surface from the list below, or propose your own if you are actively building one. The exercise works for any of these:

- **A. Coding assistant for internal data-engineering.** Users write SQL, Python (pandas / PySpark), and DAG-config YAML. The assistant completes code inline in an IDE, answers chat questions about code, and explains error messages.
- **B. Customer-support triage bot.** Users ask natural-language questions about a SaaS product. The bot retrieves from public documentation and internal KB articles, answers or escalates. Refusal on out-of-scope is required.
- **C. Document-QA app for legal contracts.** Users upload PDFs and ask questions. The app must cite the specific clauses. Hallucinated citations are the highest-severity failure.
- **D. Spreadsheet copilot with screenshot input.** Users paste screenshots of dashboards or upload sheets and ask questions or request formulas. The assistant reads charts, tables, and cells, and produces text or formula outputs.
- **E. Multi-lingual meeting-notes summarizer.** Users upload audio (or text transcripts); the assistant summarizes. Coverage across languages and topic domains is required.
- **F. Your own product.** If you work on an evaluated ML product, use it. This is the highest-value option; the templated ones above are for people who don't have their own.

State your choice at the top of the report and *why* you chose it. If you chose your own, describe the surface in one paragraph.

## Requirements

### Part A — the surface enumeration

Ship `docs/01_surfaces.md` with one paragraph per user surface, following the Chapter 7 Step 1 pattern. Focus on the *user interaction shape*, not the technology.

For product B, this would be something like:

> **Docs-grounded QA.** User arrives on a support-chat surface, types a natural-language question about the product. Bot retrieves top-5 passages from a corpus of ~4,000 public documentation pages plus ~2,000 internal KB articles. Bot generates an answer with inline citations to retrieved passages. Bot may refuse ("this looks like an account-specific question — please contact support") and offer to escalate; refusal is expected on out-of-scope questions.
>
> **Multi-turn clarification.** ...
>
> **Ticket triage handoff.** ...

Aim for 2–5 distinct surfaces.

### Part B — the capability map

Ship `docs/02_capability_map.md` with a table. Rows are capabilities; columns are (surface(s) served, weight in aggregate, instrument, benchmark/slice used).

Example row for product A:

| Capability | Surface(s) served | Weight | Instrument | Benchmark / slice |
| --- | --- | --- | --- | --- |
| Python code completion | Inline completion, chat edits | 0.35 | pass@1 with sandbox (Ch. 2) | HumanEval+ (public anchor) + 300 bespoke Python-in-data-eng items |
| SQL correctness | Chat edits, chat explain | 0.20 | Execution against a fixture DB | 200 bespoke queries against a Northwind-shape schema |
| Code explanation quality | Chat explain | 0.10 | Rubric-graded judge (mod-105) | 100 bespoke items with reference explanations |
| Instruction following | All | 0.10 | IFEval subset (mod-104) | 300-item IFEval |
| Safety / refusal | All | 0.15 | Refusal rubric (mod-109 pending) | 100 items across policy categories |
| Latency, cost | All | 0.10 | Per-turn P50 / P95, per-turn tokens | Same items as above |

The point is not the specific weights; it is that every weight is *justified* in Part C.

### Part C — the weighting rationale

Ship `docs/03_weighting.md` explaining, per capability:

- Why it got that weight. Traffic estimate, impact ranking, or hybrid.
- What traffic data (or plausible-estimate reasoning) supports the number.
- What would move the weight up or down.

Sanity check: weights should sum to 1.0. If they don't, either you have an unweighted safety axis (a separate pass/fail gate — see Part E) or you have an error.

### Part D — per-instrument configurations

Ship `docs/04_configurations.md` with per-instrument details. For each instrument you named in Part B, specify:

- Which harness (from mod-104).
- Decoding params.
- Judge model + version (if applicable).
- Human-calibration slice size and κ target.
- Sandbox / timeout policy (for code instruments).
- Extraction convention (for math or answer-extraction instruments).
- Chunker / retriever / `k` (for RAG instruments).
- Image preprocessing (for multimodal instruments).

This is the section that makes the suite *reproducible*. If a reviewer wanted to spin the whole thing up, this is where they'd find every knob.

### Part E — sentinel items

Ship `docs/05_sentinels.md` with 10–20 hand-authored items that block a release on failure. For each:

- **The item** (question, expected behaviour).
- **Why it's a sentinel.** Past incident? Legal-compliance requirement? Something that would embarrass the product? A specific reasoning failure the model has historically shown?
- **Pass criterion.** Exact behaviour required to pass.

Sentinels are not part of the aggregate. They are pass/fail. If any sentinel fails, the release is blocked regardless of aggregate score.

Example sentinels for product C (legal contracts):

- "Given a contract with no arbitration clause, ask 'What is the arbitration clause?' — assistant must not fabricate a clause. Pass = states explicitly that there is no arbitration clause."
- "Given a mutual NDA, ask 'What is the term of confidentiality?' — assistant must cite the specific clause number. Pass = citation is to the correct clause."

### Part F — the coverage-gaps section

Ship `docs/06_coverage_gaps.md`. For each gap:

- **What is not measured.** Be specific — "long-context math reasoning above 8k tokens," "code generation in Rust or Go," "questions asked in Portuguese," "screenshots of iPad-native apps."
- **Why it is not measured yet.** Cost, lack of a good instrument, product doesn't have significant traffic in that surface, etc.
- **What would happen if a regression occurred there.** Real user impact estimate.
- **Plan.** Add an instrument next quarter, monitor via a manual periodic check, or accept the gap explicitly.

This section is the *most important* one for a reviewer's trust. An eval suite writeup that implies it covers everything is one that will be blindsided; an eval suite that names what it does not cover has already reduced the blast radius of the eventual regression.

### Part G — the reproducibility manifest

Ship `docs/07_reproduction.md`. Not a placeholder — a real command list.

- Model versions and providers.
- Dataset commits / package versions for every benchmark.
- Judge versions and prompts.
- Harness versions.
- Seeds.
- The exact commands to rerun each instrument.

### Part H — the top-level report

Ship `SUITE.md` (≤ 2 pages) that summarizes the seven documents above with links. This is the entry point. A reviewer reads `SUITE.md` first and follows links to the details.

## Bundle

- `docs/01_surfaces.md`
- `docs/02_capability_map.md`
- `docs/03_weighting.md`
- `docs/04_configurations.md`
- `docs/05_sentinels.md`
- `docs/06_coverage_gaps.md`
- `docs/07_reproduction.md`
- `SUITE.md`

## Starter guidance

- **Do not run any of the instruments.** This exercise is a design exercise. You are drafting *the suite*, not executing it. Executing anything from `docs/04_configurations.md` is a separate cost line-item; the point is to have the design defensible enough to be worth executing.
- **Pick a product surface you know something about.** The templated options (A–F) are calibrated for someone building in the space. If you don't build in the space, spend 15 minutes reading about a well-documented example (Cursor, Windsurf, Perplexity, Notion AI, whatever) and use that. The concrete-surface exercise is orders of magnitude better than an abstract "an LLM product."
- **The weights must sum to 1.0.** Or you must explicitly separate the pass/fail safety axis and say so. An unnormalized set is a design error, not a stylistic choice.
- **Justify each weight in one sentence minimum.** A weight without justification is a placeholder. If you cannot say why "code completion = 0.35 and code explanation = 0.10," you have not thought about the product; go re-read your Part A.
- **Sentinels are the fastest way to make the suite useful today.** Even without any of the other machinery running, a set of hand-authored sentinels run against a candidate release blocks embarrassment. Take them seriously; do not dash off 5 generic ones.
- **The coverage-gaps section is where you build trust.** Reviewers who see an eval suite that implies coverage of everything they might care about will not trust it. Reviewers who see an eval suite that names three specific things it does not measure will trust it.
- **Do not treat this as a "just for the exercise" doc.** The pattern in Chapter 7 is one you'll ship — even the artifact structure (`SUITE.md` + `docs/*.md`) is close to what a real product would check in.
- **Refer to Chapter 5's "alert on composite, debug on components" rule.** The weighted aggregate is your dashboard number. The per-capability numbers are what you debug against. Say so in `SUITE.md`.
- **You do not need to invent numbers for capabilities you have not run.** Placeholders like `(TBD, target κ ≥ 0.7)` are fine and are the correct posture. Do not fabricate metric values.

## Acceptance criteria

- A concrete product surface is stated with a paragraph of context.
- 2–5 user surfaces are enumerated.
- The capability map has at least 5 rows with instrument, weight, and benchmark/slice named for each.
- Weights sum to 1.0 or the pass/fail safety axis is explicitly separated.
- Every weight has a one-sentence justification.
- Each instrument named in the map has a configuration entry in `docs/04_configurations.md`.
- 10+ hand-authored sentinels with pass criteria are shipped.
- The coverage-gaps section names at least three specific gaps with reasoning.
- The reproduction manifest lists model versions, dataset commits, judge versions, harness versions, and rerun commands.
- `SUITE.md` links to all seven docs and states the "alert on composite, debug on components" discipline.

## Stretch goals

- **Traffic-weighted variant.** If you have access to real production traffic (or plausibly-simulated), sample 1000 turns and hand-classify them into capabilities. Recompute the weights from empirical fractions. Compare to your intuitive weights; the delta is your intuition-vs-reality gap.
- **Impact-weighted variant.** For the same suite, produce a second weighting based on "worst-case severity if this capability regresses." Compare the two weightings. Products with clear safety failure modes look very different under impact weighting than under traffic weighting.
- **A red-team run of the sentinels.** Actually run the sentinels against a live model (whatever you have access to). Report pass/fail. This turns the design exercise into a working smoke test; the paired solutions repo will do this at scale, but a manual first pass is often the highest-signal 30 minutes you can spend.
- **Cost estimate.** For each instrument, estimate the per-release cost (tokens × turns × judge overhead + human eval time). Sum to a per-release eval cost. This is the number the finance team will ask about; having it in the writeup makes the suite defendable in a resourcing conversation.
- **Anti-regression alert rules.** Add a section defining the per-capability regression thresholds that fire alerts and the aggregate threshold that gates the release. Two thresholds — a warn and a block — are typical.
- **Cross-team review.** Send `SUITE.md` (with docs) to someone who works on a different ML product. Ask them for one thing that would be measured that the suite does not name. Add it to the coverage-gaps section (either with a plan to close it or an explicit acknowledgment). This is the empirical version of "the writeup has to survive a reviewer who wasn't in the room."
