# exercise-04: Build-vs-Buy Platform Matrix and Budget Defense

**Estimated effort:** 2 hours

## Objective

Produce a build-vs-buy platform dossier per Chapter 5, together with a budget defense per Chapter 6, for a specific organization's eval-program shape. The two artifacts share substrate — the same organization, the same eval workload, the same platform decisions — and compose together into leadership-consumable form. The deliverable is the dossier and the defense as two versioned documents, plus a spreadsheet or code substrate that computes the ranged three-year TCO and the sensitivity analysis.

## Prerequisites

- mod-112 Chapter 5 (build-vs-buy) and Chapter 6 (budgeting).
- exercise-01, exercise-02, and exercise-03 outputs — the plan defines what the eval workload does, the card defines what evidence the workload produces, and the crosswalks define the coverage the workload has to maintain. Without these, the dossier and the defense are unmoored.
- Enough familiarity with at least two of {Arize Phoenix, Langfuse, W&B Weave, LangSmith, Braintrust, Patronus, OpenAI evals} to describe what each covers and what it does not. Reading each product's docs page and one recent independent comparison suffices.
- Basic spreadsheet or Python skills for the TCO substrate.

## Requirements

### Part A — Organization context

Author a `Section 1: Context` document (0.5–1 page) describing the organization the dossier and the defense will address. Cover:

- **Eval workload.** Number of models under active development, launch cadence per model family, the shape of the workload split (research sweeps, release-blocking regression, interactive dev, scheduled monitoring, safety-canary). Rough counts, not fabricated precision.
- **Governance and regulatory posture.** From exercise-03: which regimes apply, which coverage commitments constrain the space, which data-residency requirements matter. If working with a hypothetical organization, be explicit about that.
- **Platform-team honest assessment.** Headcount available to own an in-house build, operational maturity (SRE discipline, incident-response process, on-call rotation), and existing eval infrastructure that would be migrated or replaced.
- **Launch-cadence coupling.** Chapter 6's frame: what the release train's ceiling is, and how that ceiling constrains the platform's wall-clock SLO.

### Part B — Options considered and per-layer decisions

Produce `Section 2: Options` and `Section 3: Per-layer decisions`. For at least four eval-platform layers (registry, orchestration, warehouse, and either observability or CI integration):

- Name the concrete options for that layer (in-house, OSS self-hosted, hosted vendor). Do not enumerate every vendor; pick two to four representative ones per layer.
- Score each option against the six decision axes from Chapter 5: TCO, governance posture, coverage of the five-layer surface, exit and migration cost, staffing and operational-maturity fit, roadmap and vendor risk.
- Make a per-layer recommendation (build, buy, or hybrid) and defend it by citing the axes that drove the decision.

The recommendations do not need to be uniformly "in-house" or uniformly "buy." The point of the discipline is that different layers have different economics; the exercise expects a mixed answer.

### Part C — Three-year ranged TCO

Produce `Section 4: TCO` with a spreadsheet or Python substrate:

- Three-year TCO per layer, ranged low / expected / high.
- Total three-year TCO across layers.
- Sensitivity analysis on the two most-uncertain inputs (usually: workload growth rate and vendor token unit price).
- A table showing what the TCO becomes under two alternative option choices (e.g., "if we chose full in-house for the registry instead of the recommended hybrid, the three-year TCO shifts by $X and the staffing implication is Y").

The TCO substrate is machine-runnable — a reader who wants to update the workload projection or the vendor unit price can re-run the substrate and see the number move.

### Part D — Governance and exit risk register

Produce `Section 5: Risk register`:

- Data-residency posture per bought layer, including any hosted vendor's regional coverage.
- Exit-cost estimate per bought layer — what it would take to move off it in year three if the vendor's product regressed or was discontinued.
- Vendor-risk score per bought layer, with rationale.

### Part E — Staged adoption plan

Produce `Section 6: Staged adoption`:

- The staged sequence from Chapter 5 (Stage 0: baseline; Stage 1: highest-leverage bought layer; ... Stage 5: as needed).
- Per stage: what ships, kill criteria (measurable outcomes that trigger re-open), review cadence, and the specific rollback-to-prior-stage plan if the stage misses its kill criteria.

### Part F — Recommendation and sign-off

Produce `Section 7: Recommendation`:

- The overall recommendation for the layered platform build, with a named review date.
- A named sign-off list (which roles need to sign, per Chapter 5's guidance that the compliance function co-signs for organizations with regulatory exposure).
- A one-page executive summary at the top of the whole dossier — leadership reads the summary; the details are for the platform team and the compliance co-signer.

### Part G — Budget defense composed with the dossier

Author a separate `BUDGET_DEFENSE.md` per Chapter 6, keyed to the same organization:

- **What the eval program buys.** One or two specific concrete examples (an incident the program caught, a launch it blocked with cause). No overreach.
- **Where the program sits on the frontier.** Per-workload-class shape (coverage-first vs. wall-clock-first vs. cost-first) with justification.
- **What the trade-offs are.** A description of what would change under two alternative frontier shapes.
- **What the ask is.** The specific numbers, ranged, with assumptions cited. The numbers reconcile with the TCO substrate from Part C — the two documents share a source of truth.
- **Reallocation levers.** Both directions: what would be cut if the constraint tightens, what would be added if the constraint loosens. At least three specific levers in each direction.

### Part H — 400–600-word integration note

Alongside the two artifacts, write a short note explaining:

- Where the dossier and the defense compose (the same TCO, the same workload, the same governance posture).
- What the highest-risk uncertainty in the plan is and what would trigger a re-open.
- Which decision, if reversed, would most change the platform's shape (a "what if" analysis).
- What you would tell the compliance function about the plan's implications for exercise-03's crosswalks — specifically, whether any of the platform choices affect the eval program's ability to satisfy a regime clause.

## Starter guidance

- **Do not enumerate every vendor.** Two to four options per layer is enough; a survey of every product is not the point.
- **The hybrid answer is usually correct.** Do not force a pure build or a pure buy for the whole platform. Chapter 5's frame is per-layer; the exercise expects a hybrid.
- **TCO must be ranged.** A point estimate is more precise than the underlying uncertainty warrants and less defensible under leadership questioning.
- **Governance posture can veto TCO.** For a heavily-regulated organization, data-residency requirements can eliminate options with lower TCO. The dossier should be honest about that.
- **The compliance function is a co-signer for regulated organizations.** The sign-off list is not just the platform team's leadership; the crosswalk artifact of exercise-03 has stakeholders too.
- **The defense reconciles with the TCO.** A budget defense whose numbers do not match the TCO substrate is a defense that will not survive its first finance review.
- **The staged plan has kill criteria.** A staged plan whose stages do not have measurable outcomes is a plan that will never trigger a re-open, which is how in-house builds run for years without shipping.
- **The whole artifact has a review date.** Chapter 5 argued for it; Chapter 6 amplified it. Both artifacts refresh annually at minimum.
- **Do not build the platform.** This exercise is authorship of the decision documents, not implementation. mod-111 exercises are the implementation.

## Acceptance criteria

- Section 1 (Context) is complete on the four fields, honest about the platform-team's staffing, and cites exercise-03 for the governance posture.
- Section 2 (Options) names at least two options per layer for at least four layers, and rates each on the six axes.
- Section 3 (Per-layer decisions) makes a recommendation for each layer with axis-cited justification. The recommendations are not uniformly one option.
- Section 4 (TCO) is a runnable substrate, produces a ranged three-year total, and includes sensitivity analysis on two inputs.
- Section 5 (Risk register) covers data residency, exit cost, and vendor risk per bought layer.
- Section 6 (Staged adoption) has at least three stages with kill criteria per stage.
- Section 7 (Recommendation) has a one-page executive summary, a review date, and a sign-off list including the compliance function for regulated cases.
- `BUDGET_DEFENSE.md` is present, structured on Chapter 6's four load-bearing paragraphs, and reconciles with the TCO substrate.
- The reallocation-levers section lists at least three levers in each direction (cut and grow).
- The integration note is present and does not just paraphrase the two artifacts.
- The dossier and the defense are versioned and have review dates.

## Stretch goals

- **Real-world calibration.** If the reader has access to real vendor pricing (a public price list or an anonymized quote), use it in the TCO. Note where the calibration comes from.
- **Multi-scenario TCO.** Extend the TCO substrate with a scenario axis: "conservative growth," "expected growth," "aggressive growth." Show how the recommendation changes under each.
- **Composition with exercise-03.** Add a small script that reads the crosswalks from exercise-03 and flags any dossier decision that would leave a regime clause without evidence. The script is the automation of the Chapter 4 discipline — "budget cuts that would drop below crosswalk coverage" get named at design time, not at post-cut audit time.
- **Change-of-scale replay.** Author the dossier and defense for the same organization at a different scale (10× the workload, or 0.3× the workload). The exercise is a check on the elasticity of the reasoning — if the recommendation flips completely under a modest scale change, that is signal about the fragility of the argument.
- **Leadership presentation deck.** Convert the executive summary into a five-to-eight-slide deck a VP would present to the CFO. The slide deck is the delivery vehicle; the artifact is the receipt.
- **Vendor-inquiry template.** Alongside the dossier, author a short list of pointed questions to send to each hosted vendor whose option was seriously considered. The questions are the artifacts of the axes: "what is your data-residency story in the EU?" "what is your export format for eval definitions?" "what is your incident-disclosure obligation?" The template is what a mature procurement conversation runs from.
