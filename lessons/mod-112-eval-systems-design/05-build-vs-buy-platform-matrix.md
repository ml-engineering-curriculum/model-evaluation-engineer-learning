# Build vs. Buy Across Eval Platforms

The fourth governance artifact is the build-vs-buy dossier. Its audience is leadership — the VP or C-level executive who signs off on the eval program's budget, plus the platform-team engineering director who owns the substrate. The failure mode it prevents is the *recurring rebuild conversation*: every eighteen months a new team lead or a new vendor pitch reopens the question of whether the organization should be running its own eval platform or moving to a hosted alternative, and each time the conversation is had without shared vocabulary, without agreed-on decision criteria, and without a written record of the last decision or its rationale.

The dossier is not a vendor-selection RFP, and it is not a monolithic build-or-buy decision. Its shape is closer to a *decomposition* of the eval-platform surface into components, an evaluation of each component against a small set of decision axes, and a recommendation — per component — of build, buy, or a hybrid. The output is a document leadership can consume, sign off on, and re-open in a year with the shared context intact.

The chapter walks the space in three passes. First, the platform options and what each one actually is. Second, the decision axes and how to compute a defensible score along each. Third, the staged-adoption pattern that keeps the decision reversible when the answer is initially wrong (as it will occasionally be). The exercise-04 deliverable is the dossier itself for a specific organization.

## The five platform options (and the sixth: hybrid)

"Build vs. buy" in eval-platform practice does not resolve to two options. In 2026 the meaningful space is at least six: full in-house build, and four to five distinct hosted or open-source-hosted alternatives, each of which occupies a specific niche.

### Full in-house

The organization builds every layer of mod-111 itself: the registry, the multi-runner orchestration plane, the cost controls, the warehouse, the CI integration. The platform team's investment is the whole surface; the vendor dependency is only on the model-serving APIs and the underlying compute.

When it fits: the organization has strong platform-engineering muscle, high data-residency or governance constraints that make hosted platforms non-viable, and a large enough eval workload that the amortized cost per team is competitive with hosted per-seat pricing. Google, Anthropic, OpenAI, Meta, and (increasingly) the large enterprises with dedicated ML-platform teams sit in this category.

When it does not fit: small or mid-sized organizations without the platform-engineering headcount to sustain a five-layer substrate over years. The rebuild cost is the load-bearing risk — an eval platform that gets 60% of the way to production and then stalls when the founding engineer leaves is a common shape.

### Arize Phoenix

Arize's open-source observability and evaluation platform. Focus: tracing, dataset management, experimentation, and eval running with a strong LLM-observability posture. Self-hostable (Docker or Kubernetes) or consumed as a hosted Arize AX product. Deep OpenInference / OpenTelemetry integration; well-integrated with the LangChain / LlamaIndex ecosystems. Substantial dataset and experiment features; a growing catalog of built-in evaluators.

When it fits: organizations that want a self-hostable observability + eval stack with a clean OpenInference story, that already have OpenTelemetry infrastructure, and that value dataset-manageability and human-annotation UIs. Well-suited as the observability + interactive-eval half of the platform even when the release-blocking half stays in-house.

When it does not fit: organizations whose release-blocking discipline is more like the mod-111 platform than like an observability-first workflow. Phoenix is excellent at the exploratory / iteration altitude; it is less directly a substitute for a versioned-artifact registry with content-addressed immutability.

### Langfuse

Open-source LLM observability, prompt management, and evaluation. Focus: trace ingestion, prompt version control, a `Score` primitive for evaluation, and a strong self-hostable posture. Deployed via Docker / Kubernetes or consumed as Langfuse Cloud. Prompt-management surface is a notable strength — prompt revisions are first-class objects with rollout controls.

When it fits: organizations that want prompt-management and eval-scoring in the same tool, that value a straightforward self-host path, and whose eval workload is more oriented toward scoring individual traces than running large batch benchmark sweeps.

When it does not fit: workloads that live in the batch-benchmark category more than the trace-scoring category. Langfuse's evaluation surface is score-per-trace-centric; it can be pushed to serve batch evaluations but that is not its native shape.

### Weights & Biases Weave

W&B's LLM observability and eval layer, tightly integrated with the W&B experiment-tracking substrate. Focus: `@weave.op` decorator model for capturing calls, dataset versioning, evaluations that compose with W&B's broader tracking surface.

When it fits: organizations already deep on W&B for experiment tracking, that want the training-time and eval-time observability to share a substrate, and that are comfortable with the W&B commercial posture.

When it does not fit: organizations that do not have W&B as the experiment-tracking substrate. Weave is much less useful as a standalone product; its value is in composition with the rest of the W&B stack.

### OpenAI evals

OpenAI's open-source evaluation harness (github.com/openai/evals), primarily a runner and task catalog rather than a full platform. Provides task registration, model-graded (`cot_classify`, `modelgraded`) evaluators, and a broad public catalog of tasks. Not a UI, not a warehouse, not a registry — a runner and a task collection.

When it fits: as one runner among several (mod-104 Chapter 4 was explicit), specifically the one whose task catalog and `modelgraded` idioms the organization wants to reuse. Not a platform substitute; a component within one.

When it does not fit: as the substrate for release-gate governance. OpenAI evals do not carry the registry, warehouse, or SLO discipline the rest of this module has been building toward.

### Vendor-hosted turnkey eval products

A growing category: Braintrust, Patronus AI, Vellum evals, Humanloop, LangSmith (LangChain), Confident AI (DeepEval), Galileo, Fiddler, Comet Opik, and others. These offer varying combinations of trace observability, evaluation, prompt management, and eval-run management as hosted SaaS with a UI-first workflow. The category is fragmented, moves quickly, and each product has its own strengths (Braintrust's dataset iteration UX, LangSmith's LangChain-native ergonomics, Patronus's model-graded evaluators). <!-- needs-research: verify the current feature parity across Braintrust / Patronus / LangSmith / Humanloop / Galileo before treating any specific claim in an exercise deliverable as authoritative; the category has been shifting quarter-over-quarter. -->

When it fits: small and mid-sized organizations without a platform team that need the eval discipline stood up quickly, or organizations that have consciously decided to trade governance ownership for time-to-value.

When it does not fit: organizations with strong data-residency requirements that the vendors cannot honor, organizations whose regulator crosswalk (Chapter 4) requires evidence about the eval platform's own security posture that the vendor cannot supply, and organizations whose scale makes the per-seat or per-run economics uncompetitive against in-house alternatives.

### The hybrid case (which is usually the answer)

Real organizations rarely pick one option and put every eval-platform component in it. A common shape:

- **Registry:** in-house (governance requires ownership).
- **Runners:** open-source (lm-evaluation-harness, Inspect, OpenAI evals) invoked through in-house adapters.
- **Warehouse:** in-house (data-residency and lineage requirements).
- **Observability:** Arize Phoenix or Langfuse (self-hosted).
- **Interactive-eval UI:** Phoenix, Langfuse, or a hosted vendor at the edges.
- **CI integration:** in-house glue.

The hybrid answer is not intellectually messier than the pure-in-house or pure-vendor answer; it is the answer that respects the fact that different layers of the mod-111 platform have different build-vs-buy economics. The dossier's job is to make the per-layer decisions coherent, not to pretend a single vendor covers everything.

## The decision axes

The dossier evaluates each platform option (per layer, per hybrid decomposition) against a small set of decision axes. Six axes cover most defensible dossiers.

### 1. Total cost of ownership (TCO) over three years

Not a purchase price — the cost of running the option in production for three years. Components:

- **Direct cost.** License fees for a hosted platform, or the compute + storage cost for a self-hosted deployment. Per-seat and per-run pricing bounds go here; assume the highest tier likely to be needed, not the lowest.
- **Engineer-time cost.** For an in-house or self-hosted option, the fully-loaded cost of the engineers who build and maintain it. Multi-year staffing estimate at prevailing rates. For a hosted option, the (usually smaller) engineer-time cost of integrating with it.
- **Migration and integration cost.** For any option, the cost of getting current-state data and workflows into it. This is the number that most first-pass TCO estimates miss.
- **Opportunity cost.** For an in-house or self-hosted option, what the engineering team would have built with the same time if it had not been spent here. Not always quantifiable, but always worth acknowledging in the dossier.

A TCO estimate that omits any of the four is one leadership will (rightly) discount. A TCO with all four, ranged (low / expected / high) rather than point-estimated, and validated against the eval workload's projected growth, is defensible.

### 2. Governance posture

The extent to which the option satisfies the regulator crosswalk (Chapter 4) without additional work. Sub-questions:

- Where does eval data reside? For a hosted option, is the data in a region compatible with your data-residency obligations?
- Who has administrative access to the eval evidence? Can you demonstrate this to an auditor?
- What are the vendor's own certifications (SOC 2 Type II, ISO 27001, ISO 42001, EU-adequacy status for U.S. vendors)?
- What are the vendor's own incident-disclosure obligations?
- If regulatory evidence is required about the eval platform's security posture (Chapter 4's EU AI Act cybersecurity clause, for instance), can the vendor supply it in a form the compliance function can use?

Governance posture is often the axis that constrains the space hardest. Organizations with data-residency requirements the vendor cannot honor, or with GPAI systemic-risk exposure whose evidence chain requires ownership of the eval platform, effectively have "buy" removed from the option set regardless of the TCO.

### 3. Coverage of the mod-111 five-layer surface

The extent to which the option covers the five layers of the mod-111 platform (registry, orchestration, cost controls, warehouse, CI). Sub-questions:

- Registry coverage: does the option carry versioned, content-addressed artifacts for tasks, datasets, judges, and prompts? Or is versioning left to the consumer?
- Orchestration coverage: does the option run the runners the organization needs (lm-eval-harness, Inspect, internal harnesses) or only its own eval definitions?
- Cost controls: does the option enforce per-tenant token budgets, judge-tier routing, priority queues?
- Warehouse: does the option retain lineage — model hash, dataset hash, judge hash, prompt hash, decoding config, seed?
- CI integration: does the option expose a stable API that a release pipeline can call and consume an opaque verdict from?

Options with high coverage on some layers and low on others map naturally to the hybrid decomposition above. The dossier does not treat "partial coverage" as a defect; it treats it as a signal that this option belongs to a specific layer.

### 4. Exit and migration cost

The cost of moving off the option if it stops working. Sub-questions:

- Does the option export its data in a stable, open format?
- Are the identifiers (dataset hashes, run IDs, model references) portable across substrates?
- Are the eval definitions (task YAMLs, judge prompts) authored in a runner-native format, or in a vendor-proprietary DSL that would need rewriting?
- What is the incumbent cost of migration if it happens in year three?

Exit cost is the axis that separates hosted vendors most sharply. A hosted vendor whose eval definitions live in an open OSS format (Inspect, lm-eval-harness) has low exit cost; one whose definitions live in a proprietary UI-only DSL has high exit cost. The high-exit-cost option is not disqualified — sometimes the time-to-value trade is worth it — but the dossier should record the exit cost so leadership can weight it.

### 5. Staffing and operational maturity fit

Whether the organization has the personnel to run the option successfully. Sub-questions:

- Does the platform team have the headcount to own an in-house build?
- Does the organization have the operational maturity (SRE discipline, incident response, on-call rotation) to run a self-hosted OSS deployment at production quality?
- Does the vendor's account team, support model, and response SLA meet the organization's operational needs?

The mismatch here — an ambitious in-house build in an organization without the staffing to sustain it — is one of the most common failure modes of the whole discipline. A well-staffed in-house build is best; a lightly-staffed in-house build is worse than a well-configured hosted option.

### 6. Roadmap and vendor risk

The stability of the option's roadmap and the vendor's business posture. Sub-questions:

- Is the vendor's product still under active development?
- Is the vendor financially stable? Recent funding round, revenue trajectory, hiring signals?
- Has the vendor made recent breaking-change API decisions? What is their versioning discipline?
- For open-source options, is the project actively maintained? Bus factor?
- Does the option have a strong enough ecosystem that finding replacement engineers is not a bottleneck?

Vendor risk is not a disqualifier by itself; every vendor has some risk, and organizations who wait for zero-risk options never ship. It is a factor to weight; a promising early-stage vendor with the best UX in the category may be worth accepting more roadmap risk for.

The six axes give the dossier a defensible structure. Every option scores on all six; the scores are ranged rather than point; and the dossier is honest about which axes are load-bearing for the specific organization (a startup weights time-to-value; a regulated enterprise weights governance posture).

## The dossier-as-artifact structure

A concrete shape the dossier can take:

```
Section 1: Context
  - Organization's eval workload (current + projected)
  - Governance and regulatory posture (feeding from Chapter 4)
  - Platform-team headcount and operational-maturity honest assessment
  - Existing eval infrastructure and what would be migrated

Section 2: Options considered
  - Per option: what it is, what it covers, when it fits
  - Reference to the six-axis matrix

Section 3: Per-layer decisions
  - Registry: build / buy / hybrid, with rationale citing the axes
  - Orchestration: build / buy / hybrid, with rationale
  - Cost controls: build / buy / hybrid, with rationale
  - Warehouse: build / buy / hybrid, with rationale
  - Observability: build / buy / hybrid, with rationale
  - CI integration: build / buy / hybrid, with rationale

Section 4: TCO summary
  - Three-year TCO per layer, ranged low / expected / high
  - Total three-year TCO
  - Sensitivity analysis on the two most-uncertain inputs

Section 5: Governance and exit risk register
  - Data-residency posture per hybrid layer
  - Exit-cost estimate per bought layer
  - Vendor-risk score per bought layer

Section 6: Staged adoption plan
  - Which layers ship in what order
  - Kill criteria for each stage
  - Review cadence and refresh policy

Section 7: Recommendation and sign-off
  - Named recommendation
  - Named sign-off list
  - Named review date
```

Two properties of the artifact matter more than the exact section list.

- **The recommendation is defended, not asserted.** Every recommendation cites the axes that drove it. "In-house registry" is defended by the governance-posture and exit-cost axes; "Phoenix for observability" is defended by the coverage and TCO axes. A recommendation without axis-level defense is one leadership will re-open next quarter.
- **The dossier has a review date.** Not "next year" — a specific date at which the dossier is refreshed. The category moves too quickly for a five-year-static decision; a dossier without a review date is one whose defense will erode as the world shifts.

## Staged adoption: keeping the decision reversible

An organization that adopts a hybrid plan does not migrate to it in a single big-bang project. The staged-adoption pattern below is the shape that keeps the plan reversible when parts of it turn out to be wrong.

The pattern is a sequence of stages, each of which is shippable, each of which has kill criteria, and each of which can be rolled back to the previous stage without a rebuild.

- **Stage 0: baseline the current state.** Document what evaluation the organization currently does, where the data lives, and what the current TCO is. Without a baseline, the migration's success is unmeasurable.
- **Stage 1: adopt the highest-leverage bought layer first.** For most organizations this is observability (a Phoenix or Langfuse deployment as an emit-target for existing evals). Low-risk, high-value, exit-easy. Ship it in weeks, not months.
- **Stage 2: adopt the second-highest-leverage bought layer.** For most organizations this is a hosted or self-hosted trace / eval UI for interactive work.
- **Stage 3: build or replace the highest-leverage in-house layer.** Usually the registry — the load-bearing artifact of mod-111 Chapter 2.
- **Stage 4: build or replace the second-highest-leverage in-house layer.** Usually the warehouse.
- **Stages 5+**: cost controls, CI integration, other layers, in an order that maps to the specific organization's dossier.

Each stage has kill criteria: measurable outcomes that, if missed, trigger a re-open of the dossier rather than a push forward. "By six months post-stage-2 launch, at least three teams should have migrated their exploratory eval workflow" is a kill criterion; a stage that meets it advances, and a stage that misses it triggers a review.

The staged-adoption pattern is what prevents the failure mode where an organization commits to a total build in year one, discovers in year two that a bought layer would have been dramatically cheaper, and has to unwind a substantial in-house investment to migrate. Staged migration keeps the option value alive.

## Failure modes the dossier is written against

Three failure modes recur in real organizations. Each is worth naming.

### Failure mode: the vendor slide-deck decision

A vendor pitches leadership on their platform; leadership commits to it in a hallway; the eval team discovers in year two that the vendor's coverage does not fit their workload, their governance posture cannot supply the evidence the compliance function needs, and the exit cost is prohibitive. The eval team is now stuck with a substrate that was chosen without their input.

Prevention discipline: the dossier is written by the eval team, in leadership-consumable form, ahead of any vendor commitment. Vendor pitches feed into the dossier; they do not bypass it.

### Failure mode: the perpetual in-house build

The eval team commits to a full in-house platform, ships 60% of it in year one, and never ships the remaining 40%. The platform is used but is incomplete — no CI integration, no warehouse queries beyond the basics, no cost controls. Every subsequent year the missing 40% is on the roadmap and never arrives.

Prevention discipline: the staged-adoption pattern above forces a stage-by-stage commitment. If the platform team cannot ship stage 3 within its kill criteria, the dossier re-opens; maybe stage 3 is a buy, not a build, after all.

### Failure mode: the pure per-layer optimization that never composes

The dossier picks the best-in-class option for every layer independently. In production, the layers do not compose: the observability platform's trace schema does not match the warehouse's lineage schema, the runner adapters do not resolve the registry's identifiers, and the CI integration has to do glue work no one anticipated.

Prevention discipline: the dossier's per-layer decisions are made together, and the composition question — do these options actually work with each other? — is a first-class axis, not an afterthought. A composition mismatch is a real cost that the dossier should catch during the design phase.

## Guidance for the dossier author

- **The dossier is a decision artifact, not a market survey.** Leadership does not want a summary of every vendor in the category. They want a recommendation, defended.
- **Every recommendation cites axes.** No opinions. If the recommendation says "in-house registry," the axes that drove it (governance, exit cost) are named.
- **TCO is ranged, not point.** The three-year TCO number should be low / expected / high, with the assumptions that produced each named. A single number is more precise than the underlying uncertainty warrants.
- **The dossier has a review date.** The category moves; the dossier ages. A locked-in recommendation with no review date is one that will drift.
- **Compose across layers.** Best-of-breed per layer is one thing; the layers actually working together is another. Compose the recommendation.
- **The compliance function is a co-signer.** For any organization with regulatory exposure, the dossier's governance-posture section is co-signed by the compliance function. They own the audit story downstream; they need to approve the substrate upstream.
- **Update the dossier when a stage kills.** The dossier is a living artifact, not a one-shot proposal.

## Summary

The build-vs-buy platform dossier is the fourth governance artifact this module builds. It is not a single decision but a per-layer decomposition of the mod-111 substrate against six decision axes: TCO over three years, governance posture, coverage of the five-layer surface, exit and migration cost, staffing and operational-maturity fit, and roadmap and vendor risk. The option set is at least six: full in-house, Arize Phoenix, Langfuse, W&B Weave, OpenAI evals, and vendor-hosted turnkey products — and the answer is almost always a hybrid. The dossier's structure separates context from options, options from per-layer decisions, decisions from TCO, and TCO from staged-adoption planning; every recommendation cites its axes; every stage has kill criteria; every dossier has a review date. The failure modes it is written against — the vendor slide-deck decision, the perpetual in-house build, the pure per-layer optimization that never composes — recur when any of those disciplines lapses. The next chapter turns to the fifth artifact — the budget defense — where the trade-offs the platform substrate enables are quantified and defended.
