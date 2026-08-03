# exercise-03: Regulatory Mapping — NIST AI RMF, ISO/IEC 25059, and the EU AI Act

**Estimated effort:** 3 hours

## Objective

Produce a three-regime regulator crosswalk for the same product used in exercise-01 and exercise-02 (or a distinct product if the reader prefers), following the Chapter 4 discipline. The crosswalk maps the eval program's own artifacts — datasets, judges, gates, model-card sections, warehouse queries — to the vocabulary of the NIST AI Risk Management Framework (1.0 plus the Generative AI Profile), ISO/IEC 25059, and the EU AI Act. The deliverable is the crosswalk itself as three per-regime views over a single evidence table, plus a short set of remediation plans for at least two disclosed gaps.

The exercise is a crosswalk from the eval-engineering side. It is not, and should not be treated as, legal analysis; the reader's compliance function and legal counsel are the authoritative interpreters of what each regime requires. The exercise is about translating the eval program's evidence into a shape those functions can consume.

## Prerequisites

- mod-112 Chapter 4 (this chapter). Chapters 2 and 3 provide the artifacts (release-gate plan, model card, data statements) the crosswalk cites.
- exercise-01 output (release-gate plan) and exercise-02 output (model card + data statements). If these are not available, use placeholder artifacts, but the exercise is much more meaningful when composed with real prior work.
- **Primary regulatory sources**, read once (not memorized): NIST AI RMF 1.0 (2023), NIST AI 600-1 Generative AI Profile (2024), ISO/IEC 25059:2023, and EU AI Act (Regulation (EU) 2024/1689) — particularly Article 15 (accuracy, robustness, cybersecurity), Annex IV (technical documentation), and Article 55 (GPAI with systemic risk).
- mod-101, mod-102, mod-103, mod-108, mod-109, mod-110 — the modules whose measurements the crosswalk cites as evidence.
- mod-111 Chapter 2 (registry) — the substrate on which the generated-crosswalk pattern depends.

## Requirements

### Part A — The single evidence table

Produce an artifact-side evidence table that lists every eval-program artifact the crosswalk will reference. At minimum:

- Datasets (name, revision, license, contamination status).
- Judges (name, revision, backend model version, calibration).
- Gates (from the release-gate plan of exercise-01, with SLO trace).
- Model-card sections (from exercise-02) linkable by anchor.
- Warehouse queries (identified by name; the query itself is in an appendix).

Each row carries a `regime_tags` field listing which regime clauses the artifact serves as evidence for. This is the generated-crosswalk substrate: the three per-regime views are computed from this single table rather than hand-maintained.

The evidence table is stored in a machine-parseable format (YAML, JSON, or a small SQLite schema) so that a script can generate the three views from it.

### Part B — The NIST AI RMF crosswalk view

Generate `crosswalk-nist-rmf.md` covering the Measure and Manage functions. Minimum coverage:

- **MEASURE 2.3** (system performance), **2.5** (validity and reliability), **2.7** (safety), **2.8** (security and resiliency), **2.9** (explainability and interpretability), **2.11** (fairness). At least one evidence row per subcategory.
- **MEASURE 3.x** (tracking over time). At least one evidence row citing mod-110 Chapter 6 continuous monitoring and mod-111 Chapter 5 metric-over-time queries.
- **MEASURE 4.x** (feedback loops). At least one evidence row citing the plan's change-log discipline and the incident-response linkage.
- **MANAGE 1.3, 2.x, 3.x, 4.x** as walked in Chapter 4. At least one evidence row per subcategory.
- Coverage of at least four risk categories from the NIST AI 600-1 GAI Profile (e.g., confabulation, dangerous or violent recommendations, information integrity, environmental impact), each with an evidence row citing a specific mod-109 or mod-107 measurement.

Every row is complete on the five fields (regime clause, what the clause asks, eval artifact, traceability path, gap disclosure).

### Part C — The ISO/IEC 25059 crosswalk view

Generate `crosswalk-iso-25059.md` covering:

- The base 25010 quality characteristics: functional suitability, performance efficiency, reliability, security, usability, compatibility, maintainability, portability. At least one evidence row per characteristic.
- The 25059 AI-specific characteristics: functional adaptability, user controllability, transparency, robustness, intervenability. At least one evidence row per characteristic.

Rows follow the same shape as Part B.

### Part D — The EU AI Act crosswalk view

Generate `crosswalk-eu-ai-act.md` covering:

- **Article 15(1)** (accuracy, robustness, cybersecurity). At least one evidence row per property, with confidence intervals and sample sizes documented per Chapter 3 discipline.
- **Annex IV** technical documentation. Rows for every Annex IV item the eval program owns evidence for (at minimum: general description and intended purpose; risk-management system; description of performance metrics; evaluation of the AI system; cybersecurity measures; post-market monitoring plan). Items the eval program does not own (e.g., detailed description of training methodologies) are named and pointed to the owner in the model-development team.
- **Article 55** obligations (only if the product is a GPAI with systemic risk; if not, an explicit "not applicable — this product is not a GPAI with systemic risk under Article 51" statement, with the legal-counsel citation).

Rows follow the same shape as Part B.

### Part E — Gap disclosures and remediation plans

Across the three views, the crosswalk will (honestly) disclose at least two gaps — clauses the eval program does not currently satisfy. For each gap:

- The clause is named specifically.
- The remediation plan is committed: a specific action, an owner, a target date.
- The plan's cost implication is estimated (rough dollar / wall-clock / coverage impact; Chapter 6 substrate).

A crosswalk with no gaps is either extraordinary or dishonest; the exercise expects gaps to be surfaced and remediation to be planned. The disclosure discipline is the point.

### Part F — Generation script

Ship a `generate_crosswalks.py` (or similar) that:

- Reads the evidence table from Part A.
- Emits the three per-regime views from Parts B, C, and D by grouping evidence rows on their `regime_tags`.
- Refuses to emit a view if a required clause has no evidence row *and* no explicit gap disclosure. Silent gaps are the failure mode the discipline is against.
- Emits a validation report: which clauses are covered, which have gap disclosures, which are silent (a silent one is a build-time failure).

### Part G — A 400–600-word authorship note

The note answers, at minimum:

- Which regime is the most load-bearing for this specific product, and why?
- Which of the disclosed gaps is the highest-priority to remediate, and what remediation lever from Chapter 6 would you pull to close it?
- Where does the eval program measure the same underlying thing under three different vocabularies (a case where one artifact appears in all three crosswalks) and what does that tell you about the artifact's importance?
- If the reader's compliance function or legal counsel disagreed with your mapping of an artifact to a clause, what would you do next? (This is a discipline question: the crosswalk is the eval program's translation; the compliance function's determination overrides it.)

## Starter guidance

- **Read the regime texts once before writing.** Reading the actual clauses is faster than reading a summary and will produce a more defensible crosswalk. Cite by clause number, not by paraphrase.
- **Cite by version.** "NIST AI RMF 1.0 (2023)"; "EU AI Act (Regulation (EU) 2024/1689) Article 15(1)"; "ISO/IEC 25059:2023 § 5.2." Not "the framework."
- **Do not fabricate compliance.** If the eval program does not evaluate a clause, the row is a gap disclosure with a remediation plan. Do not paper over gaps with vague language.
- **The three views share the same evidence.** The point of the generated-crosswalk pattern is that adding a new eval artifact tags it once and updates all three views. Do not maintain three parallel documents.
- **The crosswalk is not a legal document.** The eval program's job is to produce evidence; the legal and compliance functions declare compliance. The crosswalk's job is translation.
- **Compose with earlier exercises.** exercise-01's gates and exercise-02's model card sections appear in the evidence rows. If you have not done the prior exercises, the rows will be shallow.
- **The crosswalk has a review date.** The regimes evolve. A crosswalk with no review date is one that will drift.

## Acceptance criteria

- The evidence table is complete, machine-parseable, and has at least one row per artifact category (datasets, judges, gates, model-card sections, warehouse queries).
- Every row in every view is complete on all five fields (regime clause, what it asks, artifact, traceability, gap).
- The NIST view covers at least the six MEASURE 2.x subcategories, the MEASURE 3.x and 4.x families, all four MANAGE subcategories, and four GAI Profile risk categories.
- The ISO view covers all eight base 25010 characteristics and all five 25059 AI-specific characteristics.
- The EU AI Act view covers Article 15's three properties, the load-bearing Annex IV items, and Article 55 (with an explicit not-applicable disclosure if the product is not GPAI with systemic risk).
- At least two gaps are disclosed with committed remediation plans (specific action, owner, date, cost estimate).
- The generation script runs from the evidence table, produces the three views, and refuses to emit views with silent gaps.
- The authorship note answers each of the four questions.
- The crosswalks are versioned and cite each regime by version.

## Stretch goals

- **Custom regime.** Extend the substrate with a fourth crosswalk view for a regime relevant to your product's deployment (ISO/IEC 42001, ISO/IEC 23894, ISO/IEC 5259, the U.K. AISI evaluations framework, HIPAA, FDA, ECOA, or a sector-specific rule). The generation script should require no changes beyond adding rows and tags.
- **Registry-native tagging.** If the reader has done the mod-111 exercise-01 versioned registry, extend that registry so that each artifact revision carries the `regime_tags` field natively, and generate this exercise's crosswalks by querying the registry rather than a standalone evidence file.
- **Gap-burndown dashboard.** Author a small analytics query over the crosswalk artifact that reports the number of gaps by regime and by category, and the burn-down of gap count over time. The dashboard is a real substrate for the compliance function to track progress.
- **Legal-review interface.** Produce a small script that emits the crosswalk in a format specifically formatted for a legal reviewer — one row per regime clause, with the artifact and traceability path in a form a non-technical reader can consume.
- **Cross-view drift detection.** Add a check that flags cases where the same artifact is tagged for a NIST clause and an EU AI Act clause but under semantically inconsistent labels (e.g., an artifact that appears as "safety" evidence in one and "quality" evidence in the other). Semantic-inconsistency detection is where the discipline goes to next as the eval program matures.
