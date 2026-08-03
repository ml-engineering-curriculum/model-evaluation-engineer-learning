# Composing a Product-Shaped Eval Suite

The five preceding chapters covered instruments — pass@k for code, extraction plus optional CoT rubrics for math, RAGAS/TruLens metrics for RAG, VQA / alignment / grounding for multimodal. This chapter is about the composition step: given a real product surface, how do you select and weight the instruments so the resulting number moves when the product gets better and does not move when the product gets worse in ways users don't feel? Framed the other way, this chapter is a case against the default failure mode of ML eval — the *leaderboard collage*, an eval suite assembled by stapling every well-known benchmark to the model-card table.

The chapter has three parts. First, why the leaderboard collage is the default and why it lies. Second, the construction pattern for a product-shaped suite. Third, the writeup discipline that makes the suite defensible to a reviewer who wasn't in the room when you built it.

## Why the leaderboard collage is the default

Take any recently-shipped foundation model's public model card. The eval table typically has 8–20 rows. Each row is a benchmark: MMLU, HellaSwag, HumanEval, GSM8K, MATH, MMMU, ChartQA, DROP, ARC-C, TruthfulQA, HumanEval+, MBPP, IFEval, MT-Bench, Arena Elo. Each column is a candidate model. The reader compares columns.

The table is genuinely useful — it lets a foundation-model consumer compare on comparable numbers. It becomes the template because the format is legible and the individual benchmarks are widely reproduced. Teams building on top of the model then reach for the same template for their internal evals, because "everyone else runs these" is a defensible-looking answer to "which evals do we run."

That is the leaderboard-collage antipattern. Two failure modes:

1. **Coverage mismatch.** The model-card table covers what the *foundation-model vendor's users, in aggregate,* need to know about — general knowledge, code, math, image understanding, refusal, instruction-following. Your product covers a specific slice of that space. A number on a benchmark that doesn't touch your slice tells you nothing about your users' experience. A high MMLU score is not a claim about your specific customer-support flow.

2. **Weight mismatch.** The model-card table is unweighted (or equal-weighted). Your product has *load-bearing* capabilities and *nice-to-have* capabilities. A model that gains 4 points on MMLU and loses 2 points on HumanEval is *not* an improvement for a coding-assistant product, even if the aggregate is up. Aggregating without weights hides regressions where they matter.

The eval-engineering fix is not "run more benchmarks." It is "map your product surface to the eval, and let the map dictate what runs."

## The construction pattern

The pattern that generalizes across the products you'll evaluate:

### Step 1 — Enumerate the user surface

Write down, in one paragraph each, the surfaces your product presents to a user. Not the tech stack; the *user interaction pattern*. Examples:

- Coding assistant in an IDE: user types partial code, model completes; user asks in chat for edits to a file; user asks for explanation of an existing function.
- Customer-support triage bot: user asks a natural-language question about the product; system retrieves from documentation; bot answers or escalates to a human.
- Spreadsheet copilot: user uploads a screenshot of a chart or sheet; asks a question or requests a modification; model responds with text or a proposed formula.
- Document-QA app: user uploads a PDF; asks questions; system responds with citations to pages/sections.

Each of those paragraphs implies *which of the four families* is load-bearing and *which slice* of each family is relevant.

### Step 2 — Map surfaces to eval families and slices

For each surface from Step 1, list:

- **Which families it touches.** Code, math, RAG, multimodal, plus the non-family axes: safety (mod-109), classical NLP (mod-103), instruction-following (IFEval-style), factuality (TruthfulQA-adjacent).
- **Which specific slices matter.** "Code" isn't enough; is it *Python data-engineering code*, or *TypeScript React components*, or *SQL queries*? "RAG" isn't enough; is the corpus a public wiki, internal engineering docs, or user-uploaded PDFs?

Concretely, the customer-support triage bot maps to:

- RAG (dominant): faithfulness, context recall against a curated set of docs; refusal-appropriateness alongside.
- Classical NLP: intent-classification per-slice accuracy (from mod-103) on the routing step, if there is one.
- Safety: refusal on out-of-scope questions; PII-handling.
- Multi-turn coherence: separate from any single-turn benchmark.

The spreadsheet copilot maps to:

- Multimodal (dominant): ChartQA, DocVQA-style tasks *plus* a bespoke slice of representative sheets.
- Math reasoning: probably not for the modelling itself, but definitely for formula generation.
- Code: formula generation if you support Excel/Google Sheets formulas.

Do this mapping *before* you go shopping for benchmarks. Otherwise you shop for benchmarks and back-fit the surface.

### Step 3 — Choose an instrument per (surface, slice)

For each (surface, slice) pair from Step 2, pick an instrument from the module chapters:

- Pass@k with sandboxed execution for any code-generation slice.
- Answer accuracy with robust extraction (plus CoT rubric on a subset) for any math slice.
- RAGAS four-metric family plus a small human gold set for any RAG slice.
- VQA-style accuracy or judge-rubric-graded grounding for any multimodal slice.
- Judge-based rubric scoring (mod-105) for open-ended text tasks.
- Classical per-slice metrics (mod-103) for any structured classifier.
- Human eval on a small subset (mod-106) for the ceiling reference.

Each instrument gets a *bespoke slice* of items sized to your product distribution, *plus* (optionally) the public benchmark row as a comparability anchor. The public row lets you compare your model choices against public reports; the bespoke slice lets you see whether the differences apply to *your* users.

### Step 4 — Weight the components

Every suite needs a weighting. Two useful patterns:

- **Traffic-weighted.** Sample your production traffic. Estimate the fraction of user turns that hit each capability. Weight the eval components by that fraction. A coding assistant whose traffic is 70% completion, 20% chat-edit, 10% explanation weights those axes 7:2:1. Recompute traffic weights per quarter — user behavior drifts.
- **Impact-weighted.** Rank the capabilities by *how bad it is when they fail*. A safety failure typically outweighs a helpfulness gain by 100× or more. Impact weighting is subjective; document the reasoning in the writeup.

The weighted-mean is the *dashboard number*. It is not the metric — the per-capability numbers are the metric. The weighted mean is a summary for release-gate decisions; the per-capability numbers are what you debug against. Mirror the mod-107 Chapter 5 discipline for composite RAG scores: alert on the composite, debug on the components.

### Step 5 — Anti-regression sentinels

A product-shaped suite lives across releases. New releases fix some things and break others. Two disciplines here:

- **Slice-level regression alerts.** Any capability whose weighted contribution drops by more than X% between releases fires an alert. This catches "we improved math but broke code" patterns that a global mean averages out.
- **Sentinel items.** A small (10–20 items) hand-curated set of *known-critical* items — items that have failed in production, items that would embarrass the product, items that touch legal-compliance requirements. These items are not part of the aggregate; they are pass/fail. A regression on any of them blocks the release.

The sentinel-item pattern is worth its weight in gold and is under-adopted. It is also the closest thing eval has to a smoke test.

## What the writeup contains

A product-shaped eval suite is an *artifact* — it is checked into a repo, versioned, and reviewed. The writeup that accompanies it has to survive a reviewer who wasn't in the room. The template:

1. **Product surfaces.** One paragraph per surface from Step 1.
2. **Capability map.** A table with rows = capabilities, columns = (surface it serves, weight in aggregate, instrument used, benchmark(s) or bespoke slice).
3. **Weighting rationale.** How the weights were derived — traffic, impact, or a hybrid — with the underlying data (traffic distribution snapshot, impact ranking).
4. **Instruments and configurations.** Per instrument: which harness (mod-104), which decoding params (mod-104 Chapter 7), which judge model and version (mod-105), which human-calibration slice, which sandbox/timeout policy (this module), which extraction convention (this module).
5. **Sentinel items.** The list of pass/fail smoke-test items with a one-line rationale each.
6. **Coverage gaps.** Explicit — the capabilities the suite *does not* cover, with a plan to close them or a rationale for accepting the gap.
7. **Reproducibility manifest.** Model versions, benchmark dataset hashes, judge versions, harness versions, seeds, and the run command.

The gap section is the most-often-missing and the highest-value. An eval suite whose writeup admits "we do not measure X, and here's what would break if X regressed" is a suite you can trust. An eval suite whose writeup implies it covers everything is a suite that will be blindsided.

## Sizing the suite: how big is enough

The tension: statistical power (mod-101) wants large `n` per component; time and cost want small `n`. Practical sizing rules that tend to work:

- **Bespoke slices per capability: 100–500 items each.** Below 100, the CI on the number is wider than any effect you'd act on. Above 500, marginal statistical power gains are small compared to the cost of maintaining the items.
- **Public benchmark anchors: full benchmark or the standard sub-split.** Reproducibility is the whole reason for including them; sub-sampling defeats the point.
- **Judge-graded metrics: a 50–100 item human-labelled slice for judge-vs-human κ.** Republish κ per release.
- **Sentinel items: 10–20.** Curated, not statistical.

A total suite of ~1,500–3,000 items across 6–10 capabilities is common for a mid-sized product; foundation-model teams operate at 10–100× this scale.

## The leaderboard-collage trap in one paragraph

If the eval suite you would ship looks like the model-card table of a foundation-model vendor, it is not measuring your product. It is measuring what the vendor wants their customers to know about their model in aggregate. The result is a suite that goes green when the model gets better at things your users never do, goes red when the vendor changes their MMLU normalisation, and stays quiet when the model breaks the two flows your product actually depends on. The construction pattern above — surface enumeration, family-to-slice mapping, instrument choice per pair, weighted aggregate, sentinel items — is the discipline that avoids that.

## Summary

A product-shaped eval suite is built by enumerating the user surfaces the product actually presents, mapping each surface to a family (code, math, RAG, multimodal, plus the non-family axes) and a specific slice within that family, choosing the correct instrument for each pair from the toolbox this module and its predecessors provide, weighting the components by traffic or by impact, and adding a small set of hand-curated sentinel items that block releases when they fail. The writeup captures product surfaces, the capability map, the weighting rationale, per-instrument configuration, sentinel items, and — most importantly — an explicit coverage gap section. The suite it produces is smaller than a leaderboard collage, is more informative about a specific product, and admits what it does not measure rather than implying it measures everything. That discipline is the module's payoff; every earlier chapter is an instrument the composition step reaches for.
