# Build vs. Buy: Crowd Platforms and In-House Annotator Teams

Everything in the previous six chapters — guidelines, pilots, agreement statistics, adjudication, gold sets, UX, attention checks — is the same regardless of who is on the other end of the labelling interface. But *who* is on the other end changes the cost, the latency, the quality floor, and the compliance surface, and the choice becomes an engineering decision every evaluation program has to make explicitly. This chapter is the tradeoff table and the decision procedure. It intentionally does not tell you which vendor to pick; it tells you what to compare them on so that the choice is defensible.

The three sourcing modes in play:

- **Paid crowd platforms** — Prolific, Scale AI (including Remotasks / Outlier), Surge AI, Mercor, Appen, Toloka, and the internal marketplaces of the frontier labs' data teams.
- **Managed in-house annotator teams** — full-time or long-contract annotators, hired directly or via a staffing partner, working exclusively on your tasks under your tooling.
- **Expert panels** — small numbers of domain experts (licensed clinicians, working software engineers, subject-matter specialists) engaged per-hour or per-project.

Most projects end up with a mix. The question is which mode owns which portion of the work.

## The tradeoff axes

Nine axes are worth naming explicitly. Every sourcing decision is a weighting of these.

**1. Cost per label.** The dollar cost of a single labelled item, including platform fees, tooling, and the amortized overhead of the review queue. Crowd platforms are typically cheapest on simple tasks. Expert panels are the most expensive by an order of magnitude. In-house teams have a fixed cost that amortizes across items, so the per-label cost falls with volume. Do not compare a crowd platform's list price to an in-house per-label cost without including the in-house team's *total* cost divided by the *actual* label volume — a full-time annotator who labels 100 items a day costs far more per label than the payroll suggests once the tooling, management, and idle time are included.

**2. Latency to first batch.** How long between starting the project and having labels in hand. Crowd platforms can turn around a small qualified batch in hours; a fully specced Scale project might take days to weeks to get through their setup and pilot; an in-house team requires weeks to hire and onboard, though a *warm* in-house team can start immediately. Expert panels vary — if the experts are on retainer, days; if you are recruiting from scratch, weeks.

**3. Sustained throughput.** How many labels per week the sourcing mode can produce once it is warm. Crowd platforms scale essentially unboundedly on straightforward tasks; in-house teams have a fixed ceiling; expert panels are bounded by the experts' calendars and are typically small-batch.

**4. Quality floor and ceiling.** The lowest and highest quality you can *reliably* achieve. Crowd platforms with good QC (qualification quiz, gold set, attention checks) reach κ = 0.6–0.7 on well-specified tasks and can be pushed higher with tight pool selection. In-house teams that have been trained on the guideline for months routinely reach κ ≥ 0.8 on the same tasks. Expert panels are the only mode that reliably produces κ ≥ 0.8 on tasks requiring domain expertise. There is no crowd substitute for the *expertise* itself, however much QC tooling you stack around it.

**5. Consistency over time.** How much annotator drift the mode produces. In-house teams have low drift once trained; the same people work on the task for months. Crowd platforms have higher drift because the pool rotates continuously and Chapter 5's gold-rotation discipline is what keeps it in check. Expert panels have low drift because the pool is small and stable.

**6. Task specialization.** How much task-specific expertise the annotators need. Crowd platforms work well for tasks that a literate general adult can do with a good guideline and half an hour of training. They struggle on tasks that require a specific credential, a specific language proficiency, or deep familiarity with a niche domain. In-house teams can be trained to specialize over time; expert panels are specialists by construction.

**7. Data residency and compliance.** Where the data physically resides during labelling and whether that satisfies the legal regime you operate under. Most crowd platforms host data on their own infrastructure (Prolific, Scale) in specific regions, and workers can be located in any of a defined set of countries. If your data is subject to GDPR with restrictions on non-EU data transfers, HIPAA for protected health information, ITAR / EAR export controls, or contractual data-residency clauses with your enterprise customers, you must verify that a candidate platform can legally receive the data — most default configurations cannot. In-house teams under your control are the flexible mode; you choose the workers' locations and the tooling's infrastructure to match your compliance surface.

**8. Vendor lock-in.** How replaceable the sourcing mode is. Crowd platforms with a good qualified pool represent a rebuild cost if you switch; in-house teams are your teams; expert panels are typically per-project and low-lock-in. Weigh this against the ease of onboarding — a crowd platform with sunk qualification effort is cheaper to run than to switch, which is a feature and a lock-in simultaneously.

**9. Auditability.** Whether you can produce, on demand, the identity of the person who labelled each item, the guideline version they saw, and the tooling state at that time. In-house teams under your tooling are the most auditable. Crowd platforms vary — some (Prolific, Scale) expose worker IDs and audit logs; others aggregate at the batch level. If your use case requires per-label auditability (a regulated industry, a legal-review pipeline), verify this explicitly.

## The four modes named by common use case

The tradeoff table maps to a small number of standard configurations that recur across evaluation programs.

**"Fast and cheap on a general task."** Prolific or a similar academic-friendly platform for tasks that non-expert readers can do — preference labelling on general dialogue, summarization quality, safety classification on non-sensitive content. Latency in hours to a day for small batches; cost per label in the low single-digit cents to a dollar depending on task complexity; quality floor around κ = 0.5–0.6 depending on QC discipline. This is where most academic pairwise-preference studies land, and it is a defensible default for the low-stakes portion of a production eval.

**"Complex tasks with vendor QC."** Scale AI, Surge AI, and similar managed platforms handle tasks that require more elaborate QC than a self-serve platform provides — multi-step annotation pipelines, RLHF preference labelling at scale, safety red-team labelling. These platforms provide their own QC layer (their own gold sets, their own worker vetting) and typically bill on a higher per-label price that includes it. Latency of days to weeks for setup; sustained throughput can be very high; quality reaches κ ≥ 0.7 on well-scoped tasks.

**"Domain expertise on demand."** Mercor, Prolific's expert pools, and specialist consultancies match you with qualified experts (licensed physicians, working software engineers in specific stacks, translators for specific languages). Costs are 5–20× a general crowd; latency is days to schedule; volume is bounded but real. This is the mode for expert-only tasks where crowd κ would not clear the bar.

**"Full-time, in-house, high-stakes."** A directly-managed team of long-contract annotators, working exclusively on your tasks, in your tooling, under your compliance and IP arrangement. Highest cost, longest ramp, best sustained quality on complex or sensitive tasks. This is where regulated-industry labelling, safety-critical training data, and any task that touches production PII typically ends up.

## The decision procedure

Given a new task, a short procedure that produces a defensible sourcing choice:

1. **Write down the task requirements against the nine axes.** How many labels per week? What quality floor? What latency? Any residency or compliance constraint? Any specialist skill requirement?
2. **Eliminate modes that cannot meet the hard constraints.** If HIPAA applies and a platform cannot process PHI, it is out. If the task requires a licensed clinician and a general crowd cannot supply one, general crowds are out.
3. **For the remaining modes, estimate cost and latency to hit the required throughput.** A total cost of ownership, not a per-label sticker price. Include tooling, review-queue time, and adjudication cost.
4. **Estimate the quality risk of each mode.** For crowds, this is the κ risk after your QC stack (attention checks, gold set, pool filters). For in-house, this is the ramp risk. For expert panels, this is the availability risk.
5. **Consider a hybrid.** Many production evals use a crowd for bulk labels and an in-house team or an expert panel for gold-set adjudication and high-stakes slices. This is often the cheapest configuration that meets the quality bar.
6. **Pilot the top choice on the task's actual data.** A vendor's marketing case study is not evidence about your task. Run a 200-item pilot on your actual data with the vendor's actual QC configuration and measure κ before committing.

The output of the procedure is a written decision that names the mode, the estimated cost per week, the expected κ range with justification, and the fallback if the primary mode fails to hit the quality bar. That written decision is the artifact a reviewer or a manager can push back on; a sourcing decision made informally cannot be re-examined.

## Cost model worked example

A cost estimate that is honest usually decomposes as:

```
Total cost per week =
    (labels_per_week * per_label_platform_fee)
  + (labels_per_week * annotator_pay_per_label)
  + (adjudication_rate * labels_per_week * per_label_adjudication_cost)
  + tooling_amortized_per_week
  + management_and_review_headcount_amortized_per_week
```

The two lines that get missed most often are the adjudication cost (which scales with adjudication rate — Chapter 5) and the management overhead (which does not scale to zero on any mode, including "self-serve" platforms). Estimating either badly makes crowd platforms look cheaper than they are on complex tasks and in-house teams look more expensive than they are on high-volume tasks.

A useful sanity check: at your expected weekly volume, at what point does the *average total cost per label* cross over between the two modes you are considering? For most tasks the crossover is somewhere between 10,000 and 100,000 labels per month; below that, crowd platforms win on cost; above that, in-house often wins on cost *and* quality.

## Data residency: the failure mode that hurts the most

The single most expensive failure mode in vendor selection is discovering, after committing, that the vendor's default configuration violates a compliance constraint you were operating under. Concretely:

- **GDPR:** most crowd platforms allow you to restrict worker pools to EU-located workers, but the default configuration often does not; and the *platform's* infrastructure (where the data is stored during labelling) is a separate question. Verify both.
- **HIPAA:** most general-purpose crowd platforms are not HIPAA-compliant. A subset (Scale AI, some enterprise Prolific configurations, some in-house teams under a BAA) are. Requires a signed Business Associate Agreement, not just a marketing claim.
- **Export control (ITAR / EAR):** dual-use technical data cannot be labelled by non-US persons under certain regimes. In-house teams under your control are typically the only defensible mode.
- **Enterprise contractual constraints:** enterprise customers commonly impose contractual restrictions on where their data may be processed and by whom. These are typically stricter than any statutory regime and require per-customer verification.

The pattern is: the vendor's *marketing* usually says they can meet the constraint; the vendor's *default configuration* usually cannot; the vendor's *enterprise tier* can, at a price. Assume the price is real and factor it into the cost model before signing.

## Common failure modes

**Choosing a crowd platform because it's cheapest per label, on a task that requires expertise the crowd doesn't have.** The resulting κ is bad, the labels are unusable, and the "cheap" cost per label becomes infinite cost per usable label. Always pilot on actual task data.

**Building an in-house team on a task whose volume never grows.** The fixed cost dominates; the per-label cost is worse than any crowd platform would have been. Pick in-house when the volume is genuinely large or the task genuinely requires it, not because "we should have our own team."

**Signing with a vendor before verifying compliance configuration.** Discovered post-signing that the vendor's default is non-compliant, spent weeks negotiating an enterprise tier, missed the project deadline. Verify compliance in the RFP, not after.

**Treating vendor lock-in as unimportant.** After 18 months on a platform, the qualified pool is the only thing that makes the labels usable. Switching cost is real and grows over time. Consider a portable qualification protocol (your own qualification quiz, your own gold set, your own guideline) so that a switch is a matter of rebuilding the pool, not rebuilding the entire evaluation program.

**Assuming attention checks and gold sets substitute for annotator quality.** They filter for engagement, not expertise. An engaged non-expert on an expert task will still produce κ that a crowd platform's own QC does not detect. The QC stack from earlier in the module is necessary but not sufficient.

## Summary

The choice between paid crowd platforms (Prolific, Scale, Surge, Mercor and their kin) and in-house or expert-panel annotators is not a single-axis decision; it is a weighting across nine axes — cost per label, latency, throughput, quality floor and ceiling, drift, task specialization, data residency, vendor lock-in, and auditability. Four standard configurations recur — cheap crowd on general tasks, managed vendor on complex tasks, expert panels on specialist tasks, in-house on high-stakes or high-volume tasks — and most production programs run a hybrid. A defensible sourcing decision writes down the requirements against the axes, eliminates modes that violate hard constraints (especially data residency, which is the failure mode that hurts the most), estimates total cost of ownership with honest overheads, and pilots the top choice on the actual task data before committing. The nine-axis discipline is what turns the vendor question from "who is cheapest" into a decision a reviewer can push back on.

That closes the module. The exercises put the six previous chapters — guideline writing, pilot iteration, agreement statistics, adjudication, gold-set rotation, and side-by-side UX — end-to-end on real data. The sourcing decision from this chapter shows up as an implicit constraint on every exercise (which annotators can you actually recruit) and as an explicit deliverable in the pilot and calibration exercise.
