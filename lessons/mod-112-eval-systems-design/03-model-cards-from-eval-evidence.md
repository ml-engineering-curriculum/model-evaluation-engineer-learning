# Model Cards Composed from Eval Evidence

The second governance artifact is the model card. Where the release-gate plan of Chapter 2 was written for the release manager on the day of the launch, the model card is written for the reviewer who consults it later — an internal review body preparing for a launch sign-off, a customer's procurement team performing a due-diligence check, a regulator conducting an audit under the EU AI Act, a downstream integrator deciding whether to build on your API. Each of these audiences asks a slightly different question of the card, and each expects an artifact shape that a well-run eval program should recognize as its own aggregated output.

Model cards were introduced formally by Mitchell et al. (2019) as a discipline for "clarifying the intended use cases of machine learning models and minimizing their usage in contexts for which they are not well suited." Six years later, the discipline is close to a norm — Hugging Face's Hub carries model cards, the Partnership on AI publishes card templates, the EU AI Act's Annex IV specifies technical documentation that is close in shape to a model card, and the NIST AI RMF's "Measure" function names model documentation among its expected outputs. This chapter is not about the definition of a model card; it is about the discipline of *composing one from your eval program's outputs* so that the card is both faithful to what the model actually is and legible to the reviewer.

The chapter has three threads. First, the sections a card contains and what evidence maps to each. Second, the split between the internal-review audience and the external-regulator audience and how one artifact can serve both. Third, the anti-patterns — the marketing card, the ceremonial card, the one-shot card — that undermine the discipline in practice.

## The model-card schema and what each section demands

The Mitchell et al. schema names nine sections, and every serious model card produced since has been a specialization of that list. This chapter uses the schema as an anchor and describes what evidence each section demands from the eval program.

### 1. Model details

Who trained the model, what the model is (architecture, size), when it was trained, what version this card describes, and how to cite the model. This is the section where the reviewer confirms that the card describes the same model they think they are reviewing.

Evidence from the eval program: the model identifier — a hash or version string that resolves through the mod-111 registry to the exact model artifact this card corresponds to. A card that identifies its model as "our latest customer-facing chat model" is a card that will diverge from reality on the next fine-tune; a card that identifies its model as `assistant-v3.2.1@sha:8fa1b0e...` is a card whose provenance is unambiguous. Every subsequent section's evidence links back to this identifier.

### 2. Intended use

The tasks the model was built to perform, the users it was built to serve, and the deployment contexts it was validated in. This is the section that translates the product spec (Chapter 2's first input) into the reviewer's language.

Evidence from the eval program: the intended-use scope from the release-gate plan (Chapter 2), with the task inventory, user description, and deployment context copied through. Copy-through matters because it enforces a single source of truth: when the plan's intended-use section changes, the card's intended-use section is updated in the same commit, not months later. The card and the plan should be diff-reviewed together on every material spec change.

### 3. Factors

The demographic, geographic, phenotypic, or instrumental variables the model's performance may vary across. This is the section that translates the mod-103 slice discipline and the mod-108 fairness measurements into the reviewer's language.

Evidence from the eval program: the list of slice variables that appear in per-slice metric evaluations, along with a note on which slices were measured and which were not. A card whose factors list includes only the slices where the model performed well is a card that will not survive external review; the discipline is to list the factors the model *should be* evaluated across (typically informed by the product spec's user description and any fairness commitments) and to disclose where the evidence is absent as clearly as where it is present.

### 4. Metrics

The measurements the eval program computed and the reasons those measurements were chosen. This is the section where the card discloses what evidence exists at all.

Evidence from the eval program: the full metric taxonomy the plan gates on, plus any diagnostic metrics the plan reports without gating. Each metric is described with its name, its source (which dataset, which judge, which prompt), its resulting number with a confidence interval, and — critically — a note on why the metric was chosen for this model's intended use. mod-101 Chapter 1's construct-validity discipline is the substrate: a metric whose construct trace to a real user-facing property cannot be articulated in the card is a metric the reviewer will (rightly) discount.

### 5. Evaluation data

The datasets the model was evaluated on. This section pairs with the "Training data" section and is the one an external reviewer most commonly scrutinizes for contamination and provenance.

Evidence from the eval program: for each dataset, the mod-111 registry reference (dataset name, revision hash), the license, the source (as recorded in the mod-102 provenance discipline), and any contamination or overlap notes with the training corpus (mod-102 Chapter 3's contamination-detection work is the source). A card whose evaluation-data section is a bullet list of benchmark names without licenses, without revisions, and without contamination notes is a card that has done the easy half of the disclosure and skipped the hard half.

### 6. Training data

Where the training data came from. Not the eval program's direct evidence, but its relationship to evaluation data is a section the eval-composed card is uniquely positioned to describe.

Evidence from the eval program: the contamination and overlap notes from mod-102, plus a pointer to the training-data documentation the model team maintains. If the training-data documentation is absent, the card discloses that fact; the discipline is to declare what is known and what is not, not to conceal gaps.

### 7. Quantitative analyses

The per-slice, per-factor breakdown of the metrics. This is the section that turns the "Factors" and "Metrics" sections into a table a reviewer can consult.

Evidence from the eval program: the mod-103 slice tables, the mod-108 fairness tables (with the FDR-controlled reporting discipline from mod-101 Chapter 6), and the disaggregated safety-eval tables from mod-109. A card whose "quantitative analyses" section is a single aggregate number per metric has skipped the disclosure that most reviewers actually consume; the whole point of the section is the per-slice detail.

### 8. Ethical considerations

Documented risks the model poses, particularly (per Mitchell et al.) risks that "were considered during model development." This is the section that translates the mod-109 red-team results and the responsible-AI review into the reviewer's language.

Evidence from the eval program: the safety-eval results with their thresholds and their post-launch commitments (rollback signals from Chapter 2 are relevant here), plus a disclosure of the categories where safety measurements were performed and any category where they were considered and declined. mod-109's dangerous-capability and jailbreak-ASR measurements land here alongside their pinned attacker suites and their ASR bounds.

### 9. Caveats and recommendations

What the reviewer needs to know that did not fit above. Deployment guidance for downstream integrators; known limitations; the process for updating the card when the model or its evaluation evolves.

Evidence from the eval program: the change log of the card itself (below), the known-limitation list assembled from the eval program's own findings (a slice where the metric is worse and the team decided to ship anyway is a caveat, not a hidden defect), and the escalation path for a downstream integrator who observes a category of failure the card did not describe.

The nine sections are not a template to fill in mechanically; they are a checklist of what a reviewer will look for. A card that omits a section without explaining why is a card that erodes trust. A card that admits "we did not evaluate this factor because our user population does not span it" is a card that has done the discipline; a card that quietly skips the factor is not.

## The dataset counterpart: data statements and dataset cards

Model cards have a sibling artifact: the dataset card, sometimes called a data statement (following Bender and Friedman 2018) or a datasheet (following Gebru et al. 2021). The eval program is a heavy consumer of datasets; the discipline the eval program owes its datasets is analogous to the one it owes its models.

An eval-authored data statement for each evaluation dataset covers, at minimum:

- **Curation rationale.** Why this dataset exists and what construct it is meant to measure. mod-102 Chapter 2 is the substrate.
- **Language, dialect, and demographic variety.** The distribution of the data along the axes that will show up in the "Factors" section of any model card that uses it.
- **Speaker or author demographics.** For datasets whose provenance is human-generated, the disclosure of who produced the data.
- **Annotator demographics and process.** For datasets with human labels, the annotator pool description, the inter-annotator agreement statistics from mod-102 Chapter 2, and the adjudication process.
- **License and permissible uses.** The mod-102 Chapter 1 license and provenance trace, unchanged.
- **Known limitations and contamination.** The mod-102 Chapter 3 contamination assessment.

A data statement per evaluation dataset is a small artifact — a page of markdown, versioned with the dataset in the registry. Its existence is what lets the model card's "Evaluation data" section be short (one line per dataset, linking out to the statement) rather than duplicating the disclosure fifty times.

Some model cards omit data statements entirely and get away with it. As datasets accumulate and reviewers get more sophisticated, the model card that pretends its datasets need no further documentation ages badly.

## Serving two audiences from one card

A model card serves two audiences simultaneously, and the discipline that keeps one artifact useful for both is to structure the card so that each audience finds what it needs without having to read the whole thing.

### The internal-review audience

The internal reviewer — the launch review body, the safety review body, the eval owner across teams, the release manager — reads the card as part of a sign-off gate. They need:

- **A single-page summary at the top** of the card, in the order of the sign-off gate's questions. What is this model? What is it for? What is the evidence that it is safe to ship? What are the known regressions and known limits? A well-designed card front-loads this summary; a well-designed sign-off checklist maps to the summary's headings.
- **Traceable links to the full eval evidence.** The summary cites specific eval runs, specific dashboards, specific mod-111 warehouse queries. The reviewer does not have to trust the card's summary — they can drill through to the ground truth.
- **The delta against the incumbent card**, when there is one. The internal reviewer is almost never comparing against the null model; they are comparing against the last shipped model. A card that includes a diff-style comparison against the incumbent (which gates got tighter, which slices improved, which regressed) is a card that answers the sign-off question directly.

The internal audience does not need the card to be beautiful. It needs it to be accurate, dense, and diff-reviewable.

### The external-reviewer audience

The external reviewer — a regulator, an auditor, a customer's procurement office, a partner integrator — reads the card as part of a decision they are accountable for outside your organization. They need:

- **A clear statement of intended use and non-uses**, phrased so that a reviewer without insider vocabulary can determine whether their deployment is inside or outside the intended scope.
- **Metric disclosure with sample sizes and confidence intervals**, not point estimates. The mod-101 discipline shows here. A card that reports "88% accuracy" without a confidence interval and a sample size is a card that will not survive an EU AI Act Article 15 review.
- **Explicit disclosure of what was not measured.** For a regulator, the categories the eval program considered and declined to evaluate are as important as the ones it did evaluate. "We did not evaluate the model on financial-advice tasks because the intended use excludes that domain" is a defensible disclosure; silently missing evidence on the financial-advice category is not.
- **A pointer to the update policy.** How often is the card updated? What triggers a new card? Who is authorized to sign off on an update? These are questions the internal audience knows the answer to already; the external audience needs them written down.

The external audience often cares more about what the card *does not claim* than about what it does. A card that overreaches — that lists capabilities without measured limits — is a card that will be treated with more skepticism than a card that discloses limits directly.

### Serving both from one artifact

Two disciplines let a single card serve both audiences.

- **Layered structure.** The card opens with the front-page summary (for the internal reviewer). The middle sections are the nine Mitchell et al. sections, expanded to the level of detail the external audience needs. The appendix carries the raw tables, the SQL queries, the confidence-interval derivations, and the warehouse identifiers. Each audience reads to the depth they need and stops.
- **Voice discipline.** The card uses declarative, plain language throughout. Marketing prose ("state-of-the-art performance," "industry-leading safety") does not belong in either audience's read; it fails the internal reviewer's accuracy check and the external reviewer's overreach check. If the model is state-of-the-art, the number in the metrics section will make that case; the adjectives are noise.

A single-artifact card is not the only design — some organizations ship a public "user-facing" card and a private "review" card. That is a legitimate choice, but it multiplies the maintenance burden and it invites divergence between the two. The layered single-artifact approach is the default the rest of this chapter assumes.

## Cards as code: versioning, storage, and update policy

A model card is a code artifact, not a wiki page. Three properties make it one.

- **Stored alongside the model.** The card lives in the model repo, in a stable path (`MODEL_CARD.md`, `docs/model_card.md`, or the shape your repo standardizes on). Every model artifact resolves to the version of the card that describes it — the mod-111 registry's model-artifact record points at the card revision.
- **Versioned.** Every material change to the card is a commit with a rationale. A structural change to the card template (adding a new section, removing one) is a versioned schema evolution; a content change to an existing section is a versioned edit. Both are diff-reviewed.
- **Programmatically composable in part.** The metric numbers, the per-slice tables, and the dataset revisions in the card are all queryable from mod-111's warehouse. The card can and should be *partially generated* — a build step queries the warehouse for the latest gated eval results and updates the metric tables; the narrative sections remain human-authored. A card whose metric numbers are hand-copied out of a dashboard will drift.

Two update policies deserve pre-declaration in the card's "Caveats and recommendations" section.

- **When does the card get a new version?** A material change to the model triggers a new card. A material change to the eval evidence — a new dataset, a re-baselined judge, a regression discovered in production — triggers a new card. A non-material change (a typo, a broken link) does not need a version bump but does need a change-log entry.
- **When does the card get retired?** Model deprecation. When the model is removed from serving, the card moves to an archived state — still queryable, marked retired, no longer describing an active deployment. The archive is not a delete; a regulator inspecting last year's incident wants to read the card that was current then.

The update policy on the card mirrors the discipline mod-111 Chapter 2's registry applies to every other artifact type. Model cards are just registered artifacts with a longer human-audience narrative.

## Anti-patterns and the disciplines that prevent them

Three anti-patterns recur in real organizations. Each is worth naming so the discipline above is understood as their alternative.

### Anti-pattern: the marketing card

The card is written by the product-marketing team from the model's greatest hits. It leads with capability claims, quotes benchmark scores without confidence intervals or sample sizes, omits the slices the model performed worst on, and describes safety evaluations that were "conducted" without disclosing thresholds or attacker suites. The card exists but does not do the work of governance disclosure.

Prevention discipline: the card is authored by the eval owner from warehouse-linked evidence, not by marketing from a slide deck. Marketing can and should have its own communications — this card is not one of them. A public-facing capability announcement can quote from the card, but the card itself does not read like the announcement.

### Anti-pattern: the ceremonial card

The card is authored the day of the launch by an on-call engineer who copies the previous release's card, updates the version string and a couple of numbers, and ships it. The nine sections all appear; none is meaningfully updated against the current model's evidence. The card exists but does not describe the current model.

Prevention discipline: the partial-generation approach above (the metric numbers, per-slice tables, and dataset revisions are warehouse-generated) makes the mechanical parts non-copiable — they will either match the ground truth or fail a build check. The narrative sections' diff-review requires the reviewer to accept that the narrative still holds; a copy-paste narrative will be flagged when the incumbent's numbers do not match the candidate's.

### Anti-pattern: the one-shot card

The card was written well for the model's initial release. It is not updated. A year later, the model has been fine-tuned twice, the eval evidence has been re-baselined, a safety category has been added to the safety-eval suite, and the card still describes the original release. The card exists but is a historical document, not a current one.

Prevention discipline: the update policy above (material model change → new card; material eval-evidence change → new card) is enforced by a periodic drift-check. A build step compares the current model artifact's identity and the current eval evidence's warehouse identity against the identities the card references; a mismatch is an alert routed to the card's owner. A stale card is an operational bug, not just a documentation debt.

### Anti-pattern: the untraceable card

The card asserts numbers, but no one can reproduce them. The metric section says "helpfulness score: 4.2/5" and the reader who wants to check finds no dataset revision, no judge revision, no confidence interval, no sample size, no warehouse identifier. The card is legible but not verifiable.

Prevention discipline: every quantitative claim in the card links to a warehouse identifier or an attached appendix that carries the raw data. mod-111 Chapter 5's four foundational queries are the ones the card's links resolve into; a card whose links resolve to a 404 fails the drift-check above.

## The card and the release plan compose

The release-gate plan of Chapter 2 and the model card of this chapter are the two artifacts most likely to duplicate each other. Two invariants keep them coherent.

- **The plan's intended-use section and the card's intended-use section are the same text, referenced from the same source.** They do not drift independently. When one is edited, the other is edited in the same commit.
- **The plan's gate outcomes flow into the card's metrics and quantitative-analyses sections.** The gates that fired, the gates that passed, and the gates that were overridden all appear in the card as evidence about the model that shipped. A gate that was overridden with an incident review appears in the card's "Ethical considerations" section as a known limitation.

The plan is the artifact of the *decision to ship*; the card is the artifact of the *fact of the shipment*. Both describe the same eval evidence, but from complementary angles.

## Guidance for the card author

- **Author from evidence, not from narrative.** Every claim in the card resolves to a warehouse entry, a registry entry, or a mod-102 provenance trace.
- **Structure for layered reading.** Single-page summary at the top; nine Mitchell-schema sections in the middle; appendix with raw tables and query references. Each audience reads to the depth it needs.
- **Partial-generate the mechanical sections.** Metrics tables, per-slice breakdowns, and dataset revisions are warehouse-generated. The narrative is human-authored and reviewed.
- **Ship a card per model version.** A single card that "describes the family" is a card that describes no specific model.
- **Ship a data statement per evaluation dataset.** The mod-102 provenance work has already done the discipline; the data statement is the disclosure of it.
- **Diff-review card edits alongside plan edits.** The two artifacts share load-bearing text; separate reviews invite divergence.
- **Track staleness as an operational bug.** A drift-check between the card's identity claims and the current model / warehouse identity is a build step, not a nice-to-have.

## Summary

The model card is the governance artifact that composes the eval program's evidence into a document a reviewer or regulator can consume. Its nine sections — model details, intended use, factors, metrics, evaluation data, training data, quantitative analyses, ethical considerations, and caveats and recommendations — each demand specific evidence from the eval program, and each links to warehouse-backed ground truth. A companion data statement per evaluation dataset carries the mod-102 provenance disclosure the model card's "Evaluation data" section links to. A single card can serve both an internal-review audience (front-page summary, delta against incumbent, diff-reviewable) and an external-regulator audience (intended-use clarity, confidence intervals, explicit non-measurement disclosure) if it is layered and voiced declaratively. The card is code — versioned, partially generated from the warehouse, and drift-checked. The anti-patterns it is written against — marketing card, ceremonial card, one-shot card, untraceable card — recur when any of those disciplines lapses. The card and the release-gate plan compose: the plan is the decision; the card is the record. The next chapter maps both artifacts to the external frames — NIST AI RMF, ISO/IEC 25059, and the EU AI Act — under which the reviewer above will interpret them.
