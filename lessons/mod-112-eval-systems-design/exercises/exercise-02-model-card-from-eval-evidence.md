# exercise-02: Model Card Composed from Eval Evidence

**Estimated effort:** 3 hours

## Objective

Compose a model card, per the Chapter 3 discipline, for the same product used in exercise-01 (or a distinct product if the reader prefers). The card compiles the eval program's evidence into the nine Mitchell et al. (2019) sections, layered so that the same artifact serves both an internal-review audience and an external-regulator audience. The deliverable is the card itself as versioned markdown, plus a data statement for at least one evaluation dataset the card cites, plus a build script that partially-generates the mechanical sections from a mocked warehouse.

## Prerequisites

- mod-112 Chapter 3 (this chapter).
- mod-112 Chapter 2 and exercise-01 output (the release-gate plan) — the plan's intended-use scope and gate outcomes are inputs to the card.
- Mitchell et al. (2019), *Model Cards for Model Reporting* — the schema this exercise implements.
- Bender and Friedman (2018), *Data Statements for Natural Language Processing*, or Gebru et al. (2021), *Datasheets for Datasets* — the schema the data-statement companion implements.
- mod-101 Chapter 4 (confidence intervals) and Chapter 5 (paired tests) — the confidence-interval discipline the card's metric section requires.
- mod-102 Chapters 1–3 (license and provenance, gold-set with IAA, contamination detection) — the substrate the data statement compiles.
- mod-103 Chapter 1 (per-slice metrics) — the substrate the card's quantitative-analyses section compiles.
- mod-109 (safety and red-team eval) — the substrate the card's ethical-considerations section compiles.
- mod-111 Chapter 5 (warehouse) — the source the mechanical sections generate from.
- Python 3.11+ with `pydantic>=2`, `pyyaml`, and enough tooling to run a mock warehouse (SQLite fixture is fine).

## Requirements

### Part A — Card structure and layered reading

Produce `MODEL_CARD.md` following the layered structure from Chapter 3:

- **Front-page summary** (one page). Model identity, intended use in one paragraph, headline metric results (three to five numbers), a diff-style summary of the delta from the incumbent card (which gates got tighter, which slices improved, which regressed), and the update policy.
- **Nine Mitchell-schema sections** in the middle, each covering the evidence identified in Chapter 3 for that section.
- **Appendix** with raw tables, warehouse identifiers, and query references.

Every section that makes a quantitative claim links to a warehouse identifier or an appendix table. No bare numbers.

### Part B — Evidence-to-section mapping

For each of the nine sections, produce the specific mapping to eval-program artifacts:

- **Model details**: model artifact identifier (a hash or version) that resolves through the mocked mod-111 registry.
- **Intended use**: copy-through from the release-gate plan's intended-use section, cited by plan version.
- **Factors**: the list of slice variables from mod-103 / mod-108 per-slice measurements; explicit disclosure of which slices were measured and which were considered and declined.
- **Metrics**: the full metric taxonomy, each metric with name, source, resulting number with confidence interval, sample size, and a two-sentence justification of why the metric was chosen (mod-101 Chapter 1 construct-validity substrate).
- **Evaluation data**: for each dataset, the registry reference (name + revision), the license, the source, and the contamination notes.
- **Training data**: at minimum a pointer to the training-data documentation the model team maintains, or a disclosure that the pointer is absent.
- **Quantitative analyses**: per-slice tables from mod-103 / mod-108 / mod-109 disaggregated evaluations, with FDR-controlled reporting per mod-101 Chapter 6.
- **Ethical considerations**: safety-eval results (thresholds, attacker suites, ASR bounds), plus post-launch commitments and any overrides applied at launch.
- **Caveats and recommendations**: the known-limitation list assembled from the eval program's own findings, the update policy, and the escalation path for downstream integrators.

Each mapping in the card is short and cites specific artifacts.

### Part C — Data statement companion

Produce `DATA_STATEMENT-<dataset>.md` for at least one evaluation dataset cited in the card, following the Bender & Friedman or Gebru et al. schema. Minimum coverage:

- Curation rationale (why this dataset exists, what construct it measures).
- Language, dialect, and demographic variety of the data.
- Speaker or author demographics (for human-generated data).
- Annotator demographics and process (inter-annotator agreement, adjudication process).
- License and permissible uses.
- Known limitations and contamination assessment.

The card's "Evaluation data" section links to the data statement rather than duplicating the disclosure inline.

### Part D — Partial-generation build script

Implement a build script (`build_card.py` or similar) that partially generates the mechanical sections of the card from a mocked warehouse:

- Queries the warehouse for the latest gated eval results for the model artifact identified in the card's model-details section.
- Updates the metric tables (with numbers, confidence intervals, sample sizes) in the card's `## Metrics` and `## Quantitative Analyses` sections.
- Updates the evaluation-data section's dataset revision list from the mocked mod-111 registry.
- Refuses to build if the model identifier in the card does not resolve to a warehouse artifact — the drift-check discipline from Chapter 3.
- Emits a build-time report of what changed since the last card build (so a diff review is scoped).

The narrative sections (intended use, factors, ethical considerations, caveats) remain human-authored and are not overwritten by the build script.

### Part E — Versioning and drift-check

Ship the card with:

- A `card_version` field, a `card_date` field, and a `describes_model` field with the artifact identifier.
- A change log at the bottom with at least three demonstrative entries (initial publication, a metric update after a re-baseline, a section addition or narrative revision).
- A drift-check script (`check_card_drift.py` or similar) that, given the current model identifier and the current warehouse state, verifies the card's identity claims are still current. The check is designed to be run in CI on a schedule.

### Part F — Two-audience walkthrough note

Alongside the card, write a 400–600-word note that walks the card and identifies:

- What the internal-review audience finds and where.
- What the external-regulator audience finds and where.
- Where the two audiences read the same content and where they diverge.
- One or two places where the card explicitly discloses what it does *not* claim (an unmeasured factor, a category the eval program declined to evaluate, a limitation the model team has documented but not remediated).

## Starter guidance

- **Start from the release-gate plan.** exercise-01's intended-use section and gate outcomes are direct inputs; do not re-derive them.
- **Author the narrative sections before running the build script.** The build script populates the mechanical parts; the narrative sections carry the load-bearing disclosure discipline.
- **Do not write marketing prose.** No adjectives that overreach. The metric section carries the strength of the claims; the adjectives are noise.
- **Explicit non-measurement is better than silent omission.** If a factor was considered and declined, say so. A card that quietly skips a factor is less credible than one that discloses the skip with a reason.
- **The data statement is not optional for at least one dataset.** The card's "Evaluation data" section will be shallow without at least one accompanying data statement; the exercise's grade depends on both.
- **Version the card as code.** The card lives in the model repo (or its mock); a card that lives in a wiki misses the change-log discipline the exercise is teaching.
- **Do not build a UI.** The card is markdown; the mechanical build is a script. A UI is real-project affordance and not the point.

## Acceptance criteria

- The card is layered: front-page summary, nine Mitchell-schema sections, appendix. Each section is populated with evidence from the eval program's substrate (real or mocked).
- Every quantitative claim in the card links to a warehouse identifier or an appendix table with the raw numbers. No bare numbers.
- The card explicitly discloses at least one factor / category / slice that was considered and declined; the disclosure is documented.
- At least one data statement is produced and linked from the card's "Evaluation data" section.
- The partial-generation build script is present, runs against the mock, updates the mechanical sections in place, and refuses to build if the model identifier does not resolve.
- The drift-check script is present and returns a clear pass/fail signal.
- The card is versioned with a populated change log of at least three demonstrative entries.
- The two-audience walkthrough note is present and does not simply paraphrase the card.
- Marketing prose is absent from every section.

## Stretch goals

- **Multi-audience split.** Author both a "public" card (published to end users) and a "review" card (for the launch review body), and defend the discipline that keeps them from diverging. Chapter 3 argued for the single-artifact approach; this stretch goal is a defensible argument the other way for the reader who wants to make it.
- **Regime-tagged card.** Add `regime_tags` to each card section that names which regulatory clauses the section serves as evidence for (Chapter 4 preview). exercise-03 will build the full crosswalk; this stretch goal wires the primitives into the card.
- **Card-as-registered-artifact.** Register the card itself in the mock mod-111 registry as an artifact of kind `model_card` with content-addressed immutability. Demonstrate that a formatting-only edit does not change the content hash.
- **Model-card comparisons.** Produce a diff-style comparison of the new card against the incumbent card; render the diff in a form the internal-review audience consumes (a comparison table of gate outcomes, per-slice deltas, and safety-eval changes).
- **Update-policy runbook.** Author a short runbook that names the specific event categories that trigger a new card (new model artifact, re-baselined judge, new dataset, discovered production regression), the actor responsible for each event, and the expected turnaround.
- **Dataset-provenance graph.** Query the mock registry for the transitive lineage of every dataset the card cites (dataset → source → curation history) and attach the graph as an appendix.
