# exercise-01: Release-Gate Eval Plan for a Product

**Estimated effort:** 3 hours

## Objective

Produce a complete, defensible release-gate eval plan for a specific product, following the Chapter 2 discipline. The plan translates a product specification and the associated external commitments into a versioned document with five gate categories, per-gate threshold derivations, sample-size and confidence-interval treatment, rollback criteria, and a change-log discipline. The deliverable is the plan itself in a machine-parseable format (YAML or JSON), plus a short authorship note that walks the plan section-by-section and defends the trade-offs made.

## Prerequisites

- mod-112 Chapter 2 (this chapter).
- mod-101 Chapter 4 (confidence intervals) and Chapter 6 (multiple comparisons / FDR): the plan's threshold and sample-size discipline builds on both.
- mod-110 Chapter 2 (offline regression suites and SLO-mapped gates): the plan is the artifact that sits behind the mod-110 gate mechanism.
- mod-109 Chapters 2 and 3 (safety and jailbreak evals): the safety-gate section maps to specific mod-109 measurements.
- mod-111 Chapter 2 (registry) and Chapter 5 (warehouse): the plan cites registry and warehouse identifiers, not free text.
- Access to a product specification for a real product (internal or open-source) and to any external commitments (customer contracts with fairness or safety clauses, published safety policies, regulatory obligations). If working with an open-source product, use its public documentation and its usage policy as the spec substitute.

## Requirements

### Part A — Product-spec extraction and intended-use scope

Before writing gates, produce a documented **intended-use scope** for the product, sourced from the spec and any external commitments. Structure:

- **Task inventory** — the tasks the product commits to serve, phrased as measurable behaviors. Not "helps users be productive"; specific tasks.
- **User and deployment context** — who invokes the model, through what interface, with what tools attached, under what latency budget.
- **Explicit non-uses** — the tasks the product will not attempt.
- **Behavioral commitments** — the safety-policy categories that override the general helpfulness objective.
- **External-commitment inventory** — a table listing every safety policy, contract clause, and regulation the plan will need to defend against. Each row cites the source document and version.

If the spec is missing any of the four intended-use fields, document the gap and propose language, but do not fabricate content — a plan built on a fabricated spec is a plan that fails the traceability check.

### Part B — The five gate categories

Author the plan's gate sections, one section per category, with at least three concrete gates in each. Every gate declares:

- **Gate identifier and category** (`quality`, `safety`, `cost`, `latency`, `fairness`).
- **Metric source** — dataset, judge, prompt, and (where applicable) mod-111 registry references. If the eval program is not in a real mod-111 substrate, use placeholder identifiers with a documented format (`registry://datasets/gold-set@rev:c1d2`).
- **Threshold value and derivation method** — one of the four methods from Chapter 2 (historical-baseline, externally-committed, SLA-committed, power-calibrated), with the input parameters or the source document named.
- **Confidence-interval treatment** — point estimate or lower confidence bound; sample-size floor (`n_min`).
- **Action** — `block`, `warn`, or `report`.
- **Override authority** — a specific role, not "someone senior."
- **SLO trace** — the specific external commitment, product SLA, or documented decision the gate defends. A gate whose SLO trace is empty is rejected.

Distribute the gates across the categories so the coverage matches the product's risk profile: a consumer chatbot's plan is heavier on safety and fairness than an internal developer tool's plan, which is heavier on cost and latency. Document the coverage rationale in the authorship note.

### Part C — Rollback criteria

Author the plan's rollback section with at least two **fast criteria** and at least one **slow criterion**. Every rollback clause declares:

- **Named signal** — a specific measurement (mod-110 sequential monitor, mod-110 shadow comparison, a Chapter 6 drift alert).
- **Named threshold** — the crossing value.
- **Named actor** — who is authorized to make the rollback decision (on-call, safety review body, launch owner).
- **Response window** — the time-bound within which the decision must be made.
- **Rollback target** — the specific prior model revision the traffic will land on, referenced by registry identifier.

Include an escalation path for each clause: what happens if the response window elapses without a decision.

### Part D — Versioning and change-log discipline

The plan's YAML/JSON schema includes a change-log section. Populate it with at least three fictitious historical entries (a threshold re-baseline, a new gate addition, a deprecated gate) that demonstrate the discipline. Each entry:

- Cites the version number, date, and actor.
- Names the specific field(s) modified.
- Provides a rationale traceable to a warehouse query, an incident post-mortem, or a spec update.

### Part E — Authorship note

Alongside the plan, write a 500–800-word authorship note that walks the plan section-by-section and defends the trade-offs. The note answers, at minimum:

- Why is the coverage distributed the way it is across the five categories?
- Which gates are the ones you expect to fire most frequently, and how does the plan respond to that expectation (warning bands, sample-size choices)?
- Where does the plan explicitly not gate on a category, and why?
- What are the two or three gates you were tempted to include but decided against, and what changed your mind?
- What are the plan's two or three most fragile elements — the ones a re-baseline or a spec change is most likely to break?

The note is not a summary of the plan; it is a defense of it.

## Starter guidance

- **Write the intended-use section first, and do not proceed to gates until it is complete.** The commonest failure of this exercise is a plan whose gates were written from the eval program's habits rather than from the product's actual commitments. If the spec cannot support the intended-use derivation, close the gap in writing before writing gates.
- **Every gate cites its derivation method, no exceptions.** A gate whose threshold has no computation or citation behind it is one that will not survive the first close call.
- **Do not gate on constructs you cannot measure yet.** A gate that depends on a measurement the eval program does not currently perform is a wish-list item, not a gate. Author it as a "future gate" in a separate appendix with a remediation plan.
- **Rollback is the second half, not an afterthought.** A plan without rollback criteria is a plan that has thought about the launch and not about the two-hour window afterward.
- **Use a real product.** A toy product will let you mechanically produce the artifact; a real product will make the intended-use and threshold-derivation decisions load-bearing, which is where the discipline lives.
- **Compose with Chapter 4.** If the product has any regulatory exposure, several gates should reference specific regulator clauses. exercise-03 walks the crosswalk formally; this exercise references it informally.
- **Do not build a runner.** The plan is a document; the runner (mod-111 Chapter 3) executes it. This exercise is authorship, not implementation.

## Acceptance criteria

- The intended-use scope is complete on all four fields, cites the product spec by version, and documents any spec gaps in writing.
- The external-commitment inventory lists at least one item per relevant regime (safety policy, contract clause, regulatory obligation) that the product is subject to. "None" is acceptable as long as it is defended.
- Each of the five gate categories contains at least three gates. Each gate is complete on all seven fields (identifier, metric, threshold, CI treatment, action, override, SLO trace).
- Every threshold is derived by one of the four methods and cites its inputs. No bare numbers.
- Safety gates use lower-confidence-bound treatment; quality gates use whatever is defensible on their sample sizes; both are documented.
- The rollback section contains at least two fast criteria and at least one slow criterion, each complete on all five fields.
- The plan is a versioned artifact with a change log populated with at least three demonstrative entries.
- The authorship note is present, defends the coverage distribution, names the plan's most fragile elements, and does not paraphrase the plan mechanically.
- The plan is machine-parseable — a downstream script could ingest it and produce a gate report structure.

## Stretch goals

- **CI integration stub.** Alongside the plan, sketch (in pseudocode or a short shell script) how a CI pipeline would invoke the mod-111 platform's API to run the plan, consume the verdict, and gate on it. mod-111 Chapter 6's `platform.submit_suite(...)` pattern is the reference.
- **Slice-fairness parity gates.** If the product has any fairness commitment, add a per-slice parity gate with its derivation cited to a mod-103 Chapter 4 or mod-108 measurement. The gate demonstrates the paired-metric discipline (refusal parity plus over-refusal parity).
- **Rollback drill.** Author a runbook that would be followed if the fast rollback signal fires. The runbook names the on-call actor, the decision points, the communication paths, and the post-rollback review process.
- **Cross-plan lineage.** If the organization has multiple products, sketch how a shared gate library would be structured so that a change to a shared safety gate updates every product's plan through the mod-111 registry rather than by manual sync.
- **Governance integration.** Add a "who reads this plan" section that names the specific roles (release manager, safety review body, launch owner, compliance function) who consume the plan at each release, and the process by which their reviews are recorded.
