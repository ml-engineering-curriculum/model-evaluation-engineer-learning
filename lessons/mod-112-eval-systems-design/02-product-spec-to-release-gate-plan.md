# From Product Spec to Release-Gate Eval Plan

The first governance artifact of the module is the one that has the sharpest failure mode: the release-gate eval plan. When it works, it is boring — a document that says "on the next release, these gates fire, at these thresholds, with these consequences," and the release train consumes it without drama. When it fails, it fails in one of two visible ways. Either the release ships something the eval program should have caught (a launch incident with the eval evidence quoted in the post-mortem as "measured, not gated"), or the release doesn't ship on time because the eval program produced gate signals no one had pre-committed to, and the launch review has to argue in real time about what to do about them.

This chapter is about producing the plan that prevents both outcomes. It is a specific translation discipline: a product specification is the input, and a release-gate eval plan is the output. The plan's audience is the release manager, the launch owner, the on-call incident owner, and (indirectly) the leadership that reviews the launch outcome. Every part of the plan traces to something in the product spec or to a documented external commitment; nothing in the plan is present because "we usually check this."

## The two inputs and the one output

A release-gate eval plan is composed from two ground-truth inputs. Getting either one wrong contaminates the plan; getting both right lets the plan write itself.

The first input is the **product specification** — the shipped-artifact description that the product team, the ML team, and the go-to-market team have agreed on. A serviceable spec contains at least: the intended use of the model (the tasks it is expected to perform), the intended user (who is going to interact with it and under what conditions), the not-intended uses (deployments the model is explicitly not built for), the product-level SLOs (latency, availability, cost per request), and the safety commitments (categories of behavior the product commits to refuse, categories it commits to allow). If any of these is missing, the plan cannot be written from the spec alone, and the eval engineer's first move is to close the gaps with the product owner, in writing, before proceeding. A plan built on a spec that says "the model helps users" is a plan that will not survive contact with a launch review.

The second input is the **external commitments** the organization is subject to but that live outside the product spec. These include regulatory obligations (an EU AI Act high-risk classification, an ISO/IEC 42001 management system commitment), enterprise-contract clauses (a customer whose contract requires a specific fairness or safety guarantee), and public-facing safety-policy statements (the organization's published usage policy). These live in different documents than the product spec but bind the same launch; a plan that gates only on the product spec and misses the external commitments will pass launches that violate them.

The output is the **release-gate eval plan** — a versioned document, kept in the repository alongside the code and the model registry, that names every gate the release must clear. The rest of the chapter is about the plan's structure and about the disciplines that produce each of its sections.

## Deriving intended use and gate scope

Before the first gate is written, the plan documents the **intended-use scope** it will govern. Intended use is the top of the plan's chain of provenance; every gate below it exists to defend a specific claim inside it.

Intended-use derivation from the product spec is not a paraphrase. It is a structured claim with four fields, each of which the plan must be able to point at.

- **Task inventory.** The full list of tasks the product commits to serve. "Answers questions about SaaS pricing, drafts email replies to prospects, summarizes call transcripts" is a task inventory. "Helps sales reps be more productive" is not — it is not a set of measurable behaviors.
- **User and deployment context.** Who invokes the model, through what interface, with what tools attached, under what latency budget. A model deployed as a chat assistant in a browser and the same model behind an internal analyst's IDE plugin are two different deployments even if the weights are identical; each needs its own plan.
- **Explicit non-uses.** The tasks the product will not attempt. "Does not provide legal, medical, or financial advice; does not generate imagery; does not execute code on the user's behalf." Non-uses matter because they translate into gates: an over-refusal gate on the "does not attempt code execution" clause looks different from one on the "does not provide legal advice" clause. mod-109 Chapter 2's over-refusal discipline is the substrate here.
- **Behavioral commitments.** The categories where the product's public policy or its safety commitments override the general helpfulness objective. "Refuses to generate targeted harassment; refuses to produce weapons synthesis routes; refuses to bypass its own safety instructions when instructed to." These are the entries that map directly to the safety gates below.

The plan's opening section is the intended-use scope, sourced from those four fields with citations back to the product spec. A plan whose intended-use section is a slogan is a plan whose gates cannot be defended when they fire.

## The five gate categories

A release-gate plan has five gate categories. Each category has its own threshold derivation discipline and its own override policy. Mixing categories under a single threshold policy is the failure mode the taxonomy prevents.

### Functional-quality gates

The functional gates cover the metrics whose regression is a product-quality problem: task accuracy on the internal benchmark, judged helpfulness on the production-replay set, format-compliance on structured-output tasks, retrieval-hit rate on RAG tasks. mod-110 Chapter 2's quality-gate discipline applies here directly.

Thresholds are **baseline-relative**. The incumbent model's rolling P05 on the same eval, minus a documented cushion sized to the metric's measurement noise, is the standard shape. Baseline-relative gates fire when the candidate is measurably worse than a shipping model has demonstrably been; they do not fire on a candidate that merely differs.

Override authority: the launch owner, with the eval owner acknowledging in writing that the override was reviewed. Overrides land in the mod-111 warehouse alongside the run they overrode.

### Safety gates

The safety gates cover the metrics from mod-109 whose regression is a *headline* problem: refusal rate on each policy-violating category, over-refusal rate on the benign-lookalike set, jailbreak ASR on the pinned attacker suite, and any dangerous-capability score committed to a public or contractual policy.

Thresholds are **absolute**. "Refuses ≥98% of self-harm requests" is the gate; the incumbent's number is a diagnostic but not the threshold. If the incumbent has been below the committed number, that itself is a bug the incumbent-side program should be running down, not evidence the committed number is wrong.

Override authority: not the launch owner alone. Chapter 2 of mod-110 was explicit — safety overrides route through the safety review body, and an override still generates an incident review whether or not the model ships.

### Cost gates

The cost gates cover the per-request compute cost, the token consumption on representative task shapes, and the aggregate cost projection on the release-candidate's traffic profile. mod-111 Chapter 4's per-tenant budget mechanics are the substrate.

Thresholds are **budget-committed**. The product SLA and the finance-approved cost model both name a per-request or per-user cost the release must clear; the gate encodes it. A candidate whose average tokens-per-response is 40% higher than the incumbent's does not automatically fail — it fails if the projected monthly cost breaches the finance-committed number.

Override authority: the finance owner plus the launch owner. Cost overrides frequently accompany a promise to re-fit the model or the routing policy within a documented window; the plan captures the promise as a follow-up entry.

### Latency gates

The latency gates cover the wall-clock user-facing performance: time-to-first-token (TTFT), time-per-output-token (TPOT), end-to-end latency at fixed prompt shape, tail latency percentiles. mod-110 Chapter 7's serving-benchmark discipline names these; the plan encodes the thresholds.

Thresholds are typically **SLA-committed** (an external user-visible commitment) or **product-committed** (a design-choice commitment that has not been surfaced to users but that the product owner is defending). Baseline-relative latency gates are legitimate for internal-only tools but usually not for user-facing products, because latency has hard perceptual thresholds that "just slightly slower" is not a graceful language for.

Override authority: the launch owner, with the product owner acknowledging the user impact.

### Fairness and slice gates

The fairness gates cover the per-slice metric parity constraints from mod-103 Chapter 4: performance gap between protected slices, over-refusal parity, error-rate parity, and any specific fairness commitment the product has made contractually or publicly.

Thresholds are **contract-committed** or **regulator-committed**. A contract that says "the model must perform within 5 percentage points on English and Spanish support-ticket triage" translates directly into a slice gate; the threshold is the 5-percentage-point number. Absent an explicit commitment, the fairness discipline still runs and still reports, but it does not gate — a fairness threshold "we thought sounded strict" is not a defensible blocker.

Override authority: for contract-committed gates, contract-review; for regulator-committed, the compliance function; for internal-policy, the responsible-AI review body.

The plan structures each of the five categories with its own section, its own threshold-derivation notes, and its own override-authority declaration. A plan that treats the five uniformly will over-block on quality or under-block on safety.

## Deriving thresholds without hand-waving

Every gate in the plan declares a threshold, and every threshold traces to one of four derivation methods. Which method is used depends on the gate category (above), but the four are exhaustive: any threshold that cannot be pointed at one of the four is a threshold the plan cannot defend when it fires or fails to fire.

- **Historical-baseline derivation.** The threshold is `incumbent P05 - cushion`, where the incumbent's P05 comes from mod-111's warehouse over a documented window (typically the previous 30 or 90 days of scheduled runs) and the cushion is sized against the measurement's bootstrap variance from mod-101 Chapter 4. The plan attaches the derivation script or the SQL query in the appendix; the threshold in the plan is not a number, it is a computation.
- **Externally-committed derivation.** The threshold is a number that appears in an external artifact — a signed customer contract, a published safety policy, a regulatory requirement. The plan cites the artifact by version and locates the specific clause. Externally-committed thresholds are more stable than baseline-derived ones because the ground truth is outside the eval program's control; they are also less flexible when the model is close to the line.
- **SLA-committed derivation.** The threshold is a product-level SLO that has been socialized to end users through the product's documentation or its Service Level Agreement. Latency and availability gates typically fall here; cost gates sometimes do.
- **Power-calibrated derivation.** The threshold is set from a power analysis — given the sample size, the metric variance, and the effect size the plan claims to be able to detect, the threshold is placed so that the false-blocker rate at the intended sample size is under a documented bound. mod-110 Chapter 4's MDE machinery is the substrate. This method is used most often for A/B or shadow-based gates where the metric is measured on production traffic.

The plan lists, for each gate, which derivation method produced its threshold and where the input parameters live. A gate whose threshold was set by "seemed right" is a gate that will not survive its first close call.

## Sample-size and confidence discipline

Every gate declares its **minimum-sample requirement** (the `n_min` from mod-110 Chapter 2's YAML) and its **confidence-interval treatment**. Two decisions each gate has to make:

- **Point estimate vs. lower confidence bound.** A gate can trigger on the metric's point estimate (`accuracy < 0.85`) or on the metric's lower confidence bound (`accuracy_lcb_95 < 0.85`). The choice is a policy — LCB-based gates are more conservative (harder to trip) but more defensible under noise; point-estimate gates are more sensitive but noisier. Safety gates typically use LCB (a headline-risk regression should not be dismissed as noise); quality gates typically use point estimate on adequately-powered suites and LCB on smaller ones.
- **Sample-size floor.** The `n_min` gate refuses to fire if the evaluated sample is smaller than the floor. Refusing to fire is not the same as passing; the plan requires an escalation ("not enough data to gate; escalate to eval owner") rather than a silent green. This is the mod-101 Chapter 4 discipline restated in gate-mechanics language.

A plan whose gates do not explicitly commit to a sample-size floor will produce non-reproducible verdicts on the first tail-of-the-week release when a batch job produced fewer completions than usual.

## Rollback criteria: the other half of the plan

A gate is one half of the plan. The other half is the **rollback criteria** — the conditions under which a *shipped* release is rolled back to the previous incumbent. Rollback is not the same as gating: gating operates on a candidate that has not yet reached production; rollback operates on a candidate that reached production and is now behind live traffic.

Rollback criteria have four properties every plan enforces.

- **A named signal.** Each rollback clause identifies a specific measurement whose crossing triggers the rollback consideration. "Refusal rate on the sequential monitor from mod-110 Chapter 5 drops below 0.97 with 99% confidence" is a named signal. "Users report the model is worse" is not; feedback is an *input* to the eval program that produces a named signal, not a rollback criterion itself.
- **A named threshold.** Each clause commits to the value that crosses. The threshold is often (but not always) the same as the corresponding gate; a candidate that squeaked past a gate at 0.98 refusal rate may still trigger a rollback clause at 0.96 in production, and the plan can commit to both numbers independently.
- **A named actor and a named response window.** "The on-call incident owner has one hour from alert to decide: continue, escalate to the safety review body, or roll back." Rollback is a time-bound decision, not an eventuality that will get around to being decided.
- **A named rollback target.** The prior model revision the traffic will land on, and the readiness of that revision to accept production traffic without warming up. mod-111 Chapter 5's lineage graph is the substrate that lets the plan name the specific revision.

A release plan that describes gates without rollback criteria is a plan that has thought about the launch but not about the two-hour window after the launch when a real regression is manifesting. Two common shapes of rollback criteria:

- **Fast criteria** trigger on user-visible metrics (latency, availability, refusal rate on public-facing categories) with a 15-minute-to-one-hour response window. They usually pre-authorize the on-call to roll back without further review.
- **Slow criteria** trigger on aggregate metrics computed over hours or days (win-rate on the internal judge, per-slice regression detection). They authorize an escalation to the safety review body or the launch owner, but not immediate rollback.

Both shapes should be present in a plan that expects to catch both fast and slow regressions.

## Plan versioning and the change log

The plan is a versioned document. Every edit has a change-log entry, a signed-off reviewer, and a date. Three edit categories are common enough that the plan pre-declares how each is handled.

- **Threshold re-baseline.** The incumbent's rolling P05 has shifted; the baseline-relative threshold is recomputed. The re-baseline is an *event* — it lands as a change-log entry with the old and new thresholds and the warehouse query that produced the shift. mod-110 Chapter 2's re-baselining discipline is the substrate.
- **New gate.** A category of behavior was not covered by any existing gate; a new gate is added. The change log declares the SLO trace, the derivation method, and the audience-approved sign-off.
- **Deprecated gate.** A gate is retired because its underlying construct has been superseded, its measurement has been shown to be flaky beyond usefulness, or the commitment it defended has been retired. Deprecation is a change-log entry with a link to the successor gate (if any) or to the memo that retired the commitment. Silently dropping a gate is the failure mode that lets committed protections quietly erode.

A plan without a versioning discipline will drift over time; the plan of record will diverge from the plan that fires in CI, and the launch review will ask which one is authoritative and no one will know.

## The plan-as-artifact structure

A concrete shape a plan can take (adapt to your organization's tooling; the shape is what matters, not the file format):

```yaml
plan_id: assistant-model-release-v1
version: 2026-08-01.3
model_family: assistant
intended_use:
  tasks: [...]
  users: [...]
  non_uses: [...]
  behavioral_commitments: [...]
  source: spec@rev:a1b2c3
external_commitments:
  - {kind: safety_policy, url: policy@v2.4, clauses: ["§3.2", "§4.1"]}
  - {kind: contract,      customer: acme,   clauses: ["fairness clause 4"]}
  - {kind: regulation,    reference: eu-ai-act, article: 15}
gates:
  - {gate_id: quality.helpfulness.mean,   category: quality,   ...}
  - {gate_id: safety.refusal.self_harm,   category: safety,    ...}
  - {gate_id: cost.tokens_per_response,   category: cost,      ...}
  - {gate_id: latency.ttft_p95,           category: latency,   ...}
  - {gate_id: fairness.slice_gap.en_es,   category: fairness,  ...}
rollback:
  - {name: fast.refusal_regression, signal: mod110_seq_monitor.refusal_rate, ...}
  - {name: slow.helpfulness_regression, signal: mod110_shadow.helpfulness, ...}
change_log:
  - {version: 2026-08-01.3, date: 2026-08-01, actor: eval-owner, note: "..."}
```

This is not the only plan shape, but it demonstrates the invariants: every section is grounded, every gate belongs to a category with its own discipline, rollback is present, versioning is present, and every field points at a specific external artifact.

## Failure modes the plan is written against

Four failure modes recur and are worth naming so the plan structure is understood as their alternative.

### Failure mode: the plan is a checklist, not a contract

The plan says "run these evals" without saying "these are the numbers below which we do not ship." The release train consumes the plan as a task list; the gates never really gate. The alternative is the plan-as-contract discipline above: every gate declares a threshold, an action, and an override authority.

### Failure mode: the plan gates on internal metrics with no external anchor

The plan gates on the internal judge's helpfulness score at 4.0 out of 5. When the launch review asks why 4.0, no one has a defense. The threshold was picked because it was "usually where we land." The alternative is threshold-derivation discipline: baseline-relative with a warehouse-computed number, externally-committed to a policy or contract, SLA-committed, or power-calibrated. A defenseless threshold is one that will be overridden on the first push.

### Failure mode: gate categories are conflated

The plan uses a uniform threshold policy across quality and safety, and the launch review resolves a close safety call using the same override authority that resolves close quality calls. The alternative is the five-category taxonomy: safety has its own threshold philosophy and its own override path.

### Failure mode: rollback is an afterthought

The plan is exquisite about pre-launch gates and silent about what happens after the launch. When the sequential monitor from mod-110 Chapter 5 fires two hours after the ramp, the plan has nothing to say and the on-call has to invent policy in real time. The alternative is a rollback section with named signals, named thresholds, named actors, and named response windows.

## Guidance for the plan author

- **Write the intended-use section first, and refuse to write gates until it is finalized.** The gates are downstream of the intended use; writing them first produces gates whose SLO trace is invented after the fact.
- **Every gate cites its derivation method.** No exceptions. A gate whose threshold is a bare number is a threshold nobody will defend at 2 a.m.
- **Safety gates get their own review track.** Safety threshold decisions do not travel through the same review as quality gates. Chapter 6 will re-encounter this when the platform's own SLOs are set.
- **Rollback is a first-class section.** Half the plan's real weight is post-launch.
- **The plan is a code artifact.** It lives with the model repo, it is diff-reviewed, and its history is a change log — not a wiki page.
- **Compose with mod-111.** The gate's dataset revision, judge revision, and prompt revision are all mod-111 registry references. Baseline-P05 numbers are mod-111 warehouse queries. A plan that duplicates those inputs will drift; a plan that links them stays honest.

## Summary

A release-gate eval plan is the artifact that translates a product specification and the organization's external commitments into a concrete list of gates, with pre-committed thresholds and pre-committed rollback triggers. Its inputs are the product spec (intended use, users, non-uses, behavioral commitments, SLOs) and the external commitments (regulations, contracts, public policies). Its gates fall into five categories — quality, safety, cost, latency, fairness — each with its own threshold-derivation discipline and its own override authority. Every threshold traces to one of four derivation methods; every gate declares a sample-size floor and a confidence-interval treatment. The plan's second half is the rollback criteria: named signals, named thresholds, named actors, and named response windows for the shipping-then-degrading case. The plan is versioned as code, its change log is a first-class section, and its failure modes — checklist-not-contract, unanchored thresholds, conflated categories, missing rollback — are what the disciplines above are structured to prevent. Chapter 3 turns to the second consumer's artifact: the model card, in which the plan's outputs and the eval program's evidence compose into a document a reviewer or a regulator can consume.
