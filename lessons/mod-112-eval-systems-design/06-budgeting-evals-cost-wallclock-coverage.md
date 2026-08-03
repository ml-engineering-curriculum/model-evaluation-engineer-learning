# Budgeting Evals Across Cost, Wall-Clock, and Coverage

The fifth and final governance artifact is the budget defense. Its audience is leadership again — the executive who approves the eval program's annual budget — and the finance function that will read the same document a month later when the invoice from the model vendor is 30% over the quarter's projection. The failure mode it prevents is the *budget surprise*: three years of quiet growth in vendor spend, coverage, and wall-clock without anyone documenting the trade-off, followed by a top-down mandate to cut spending in half with no shared understanding of what the eval program actually buys.

The chapter is about the three-axis trade-off — cost, wall-clock, and coverage — that every eval program navigates, whether consciously or not. It names the frontier the axes describe, walks the three canonical program shapes (coverage-first, wall-clock-first, cost-first), explains how the launch cadence couples the axes to each other, and lays out the leadership defense narrative that turns "we spend $X on evaluation" from a line item into a program leadership can invest in confidently. The exercise-04 dossier of Chapter 5 was about *what substrate* the program buys; this chapter is about *how much of it* the program uses.

## The three axes

Cost, wall-clock, and coverage are the three axes an eval program can be described along. Every eval program has a value on each; a program that has thought about the trade-off has *declared* its values on each and defends the trade-off; a program that has not is drifting on all three and will pay the drift back in a budget review.

### Cost

The direct dollar cost of running the eval program. Components:

- **Vendor tokens.** The output tokens paid to model vendors for the judged model and for the LLM judges. mod-111 Chapter 4's judge-tier routing and prompt caching are the substrate; mod-111 Chapter 5's per-tenant attribution query is the accounting.
- **Compute.** For in-house or self-hosted deployments, the compute cost of running the eval workloads themselves and (for locally-served judges) of the judge models.
- **Storage.** The eval warehouse's storage, especially the payload store — the raw prompt / response bytes are the largest single component past year one. mod-111 Chapter 5's retention split is the substrate.
- **Human labor.** The cost of the human evaluators on the panel from mod-106, the annotators on new datasets from mod-102, and the on-call time on the eval platform itself.
- **Platform-team salary amortization.** For an in-house or hybrid deployment, the fraction of the platform team's fully-loaded cost attributable to the eval program.

A well-run eval program can name each of the five components as a monthly or annual number, sourced from the mod-111 warehouse for the vendor and compute components, from a per-project ledger for the human-labor component, and from HR planning for the salary component. A poorly-run eval program can name one or two and estimate the rest with a factor-of-two range.

### Wall-clock

The time from "a candidate model is ready to evaluate" to "the eval verdict is available to the release train." Two aspects:

- **Per-release wall-clock.** How long a single release candidate takes to clear the offline regression suite. mod-111 Chapter 6's platform-side SLOs define this at the platform layer; this axis is about the eval program's *policy* commitment on that SLO. A release train that has committed to daily deploys and an eval program whose release-candidate wall-clock is 12 hours are structurally incompatible.
- **Full-coverage wall-clock.** How long the eval program's full coverage — every benchmark, every safety category, every fairness slice — takes to run end-to-end. This is a different number from the per-release wall-clock; it is the number that matters for a periodic full-audit run, a regulator-facing full-evidence refresh, or an incident-driven "run everything against this model right now" request.

Wall-clock is often the axis least explicitly declared. Programs tend to accumulate wall-clock as the coverage grows; unless the launch cadence is coupled to the eval wall-clock (below), the growth is invisible until the release train complains.

### Coverage

The scope of what the eval program measures. Coverage is the axis with the widest interpretation, and the shape a program has committed to is worth writing down explicitly. Sub-questions:

- **Task coverage.** Which task categories from the intended-use scope (Chapter 2) are evaluated? Which are not?
- **Slice coverage.** For each evaluated task, which slices (from mod-103, mod-108) are broken out for per-slice reporting? Which slices are known to exist but not currently reported?
- **Safety-category coverage.** Which of the safety categories (mod-109) are evaluated on this release? Which are evaluated only on the periodic full-coverage sweep?
- **Regime coverage.** Which of the regulatory clauses (Chapter 4) are supported by eval evidence? Which have a gap disclosure?
- **Depth coverage.** Per task, how many prompts are evaluated? Is the sample-size floor above the mod-101 Chapter 4 threshold for the confidence-interval discipline the plan requires?

Coverage is the axis that most naturally grows on its own — new tasks are added, new categories, new slices, new depths — and unless the growth is deliberate, it is the axis whose growth drives the other two.

## The frontier

Cost, wall-clock, and coverage are not independent. An eval program can improve any two of them at the expense of the third, but improving all three at once requires an actual efficiency gain in the substrate (a better judge, a cheaper runner, a faster orchestration plane). This is not a metaphor — it is a real Pareto frontier that every eval program sits on some point of.

- **Fix cost and wall-clock: coverage is bounded.** Given a fixed monthly vendor budget and a fixed per-release wall-clock ceiling, the coverage the program can run in that budget and in that time is bounded. Adding coverage requires either more budget or more wall-clock.
- **Fix cost and coverage: wall-clock is bounded from below.** Given a fixed budget and a fixed coverage commitment, the wall-clock cannot be shrunk below the physical minimum imposed by the least-parallelizable component of the runs (rate-limited vendor endpoints, sequential monitors, human-in-the-loop panels). Faster wall-clock requires either more parallelism (more cost) or less coverage.
- **Fix wall-clock and coverage: cost is bounded from below.** Given a fixed wall-clock and a fixed coverage, the cost is bounded by what it costs to run that coverage in that time. Reducing cost requires either more wall-clock or less coverage.

The frontier is real, and it moves — a new cheaper judge shifts it, a new caching primitive shifts it, a new parallelism story shifts it. The mod-111 platform's job (Chapters 3–5) is to make the frontier as favorable as possible; the eval program's job (this chapter) is to declare where on it the program is choosing to sit, and to defend the choice.

## The three canonical program shapes

Real eval programs cluster into three shapes along the frontier. Each is a defensible choice; each is optimized for a different organizational context.

### Coverage-first

The program prioritizes breadth and depth of measurement. Every intended-use task is evaluated; every slice is broken out; every safety category is on every release; the sample-size floor is above what mod-101 discipline requires for tight confidence intervals; the regulator crosswalk has few gap disclosures.

Cost: high. Wall-clock: usually high (though sometimes tolerable if the launch cadence is slow).

When it fits: regulated organizations with strong compliance exposure (an EU AI Act high-risk deployment, a GPAI systemic-risk classification, an ISO/IEC 42001 conformity claim), or safety-critical deployments where a missed regression is a headline event.

When it does not fit: fast-iterating startups whose launch cadence would strangle. A coverage-first program on a daily launch cadence is one where nobody deploys because the eval never finishes.

### Wall-clock-first

The program prioritizes short wall-clock so the release train stays fast. Coverage is bounded by what fits in the wall-clock ceiling; anything beyond the ceiling runs on a periodic (nightly, weekly) full-coverage sweep separated from the release path.

Cost: moderate to high (parallelism to hit the wall-clock target usually costs).

When it fits: fast-iterating product teams whose competitive advantage depends on rapid iteration. Consumer AI products, developer-tool products, and internal platforms whose users are the same organization as the model builders.

When it does not fit: heavy compliance workloads where "we run a subset on every release and a full sweep once a week" is not defensible to the regulator or the safety review body. It also does not fit organizations whose slow-signal safety measurements need production ramps to be exposed — the wall-clock-first program tends to under-detect those.

### Cost-first

The program prioritizes minimal spend. Coverage is bounded by what a small vendor bill supports; wall-clock is often long because parallelism is expensive.

Cost: low. Wall-clock: usually long. Coverage: usually narrow.

When it fits: early-stage organizations whose eval program is a discipline they are still establishing, or organizations whose leadership has not yet been convinced to invest in eval. The cost-first shape is not a permanent home; it is a starting point.

When it does not fit: any organization whose launch decisions materially depend on eval outputs. A cost-first eval program that gates a launch is a launch that is being gated by a program that has consciously accepted coverage gaps.

Most mature organizations sit on a hybrid — coverage-first for safety and regulator-facing evidence, wall-clock-first for functional-quality gates in CI, cost-first-adjacent for exploratory research work. The dossier for the eval program describes the hybrid explicitly rather than pretending the program is uniformly optimized.

## Launch cadence and the axes' coupling

The launch cadence — how often the organization ships a new candidate to production — is the variable that most tightly couples the three axes. Two aspects deserve explicit treatment.

- **Cadence sets the wall-clock ceiling.** If the organization ships weekly, the release-blocking wall-clock ceiling is at most a few days. If the organization ships daily, the ceiling is hours. If the organization ships hourly (as some services do), the ceiling is minutes. A wall-clock that exceeds the cadence's ceiling is either bypassed by the release train (a governance failure) or blocks it (a product-velocity failure). Neither is acceptable.
- **Cadence multiplies the cost of coverage.** A coverage-first program at a weekly cadence pays for coverage once a week. The same program at a daily cadence pays for it five times as often. The cost of coverage scales with cadence *unless* the program has consciously separated release-blocking coverage from periodic full-coverage sweeps. The separation is a design choice; unpacking it is a defense the leadership audience needs to see.

A leadership defense that talks about eval cost without acknowledging the launch cadence is defending a number in a vacuum. A defense that says "our launch cadence is weekly, our release-blocking coverage costs $X per release, our periodic full-coverage sweep costs $Y per month, and the annualized total is $Z" is defending a program.

## Reallocation levers

The frontier moves, and the program's position on it can be shifted deliberately. The levers are worth naming explicitly so leadership can be told what is available if the constraint changes.

- **Shift coverage from every-release to periodic.** Move safety-category evaluations that do not need to be on every release into a nightly or weekly sweep. Saves cost and wall-clock at the expense of catch-latency on regressions in the demoted categories.
- **Shift to cheaper judges.** mod-111 Chapter 4's judge-tier routing is the substrate. A move from Tier 3 (frontier judge) to Tier 1 or Tier 0 (cheaper judge or heuristic) saves cost proportionally, at the risk of measurement noise; the shift needs a validated agreement statistic from mod-105 before it is credible.
- **Increase parallelism.** Add reserved concurrency, batch multiple candidates in the same vendor batch job (24-hour SLA batch APIs from OpenAI, Anthropic, and Google carry ~50% discounts on the tokens they cover), pre-warm judge caches. Saves wall-clock at the cost of increased peak spend.
- **Trim redundant coverage.** Two evaluations that measure the same construct with correlation > 0.9 are candidates for one being retired; a slice-level breakout that has never once distinguished the candidate from the incumbent is a candidate for demotion. The mod-101 Chapter 6 FDR-controlled reporting discipline informs which trims are defensible.
- **Move to batched evaluations.** Instead of running every candidate individually, batch multiple candidates in the same sweep and reuse the judged-baseline computation. Saves cost with negligible impact on wall-clock at the expense of slight indirection in the per-candidate report.
- **Increase or decrease sample-size floors.** Larger samples per gate give tighter confidence intervals (mod-101) at higher cost; smaller samples reduce cost but widen CIs, which raises the false-blocker rate if the plan gates on point estimates.
- **Renegotiate vendor pricing.** As spend crosses six or seven figures a year, the vendor's account team usually offers volume discounts. This is a real lever; a program whose spend has grown 5x since the last vendor contract review is one whose vendor bill has probably drifted up more than it needs to.

The reallocation levers form the "if we had to cut spend by 30%, here is what we would do" and "if leadership wanted to double our coverage, here is what we would need" appendices in the budget defense. Both directions matter — leadership frequently changes constraints in both directions in the same year.

## The leadership defense narrative

The budget defense is not a table. It is a short document that leadership can read in fifteen minutes and act on. The narrative has four load-bearing paragraphs.

- **What the eval program buys.** The connection between the eval program and the decisions leadership itself cares about. Not "we run 200 evaluations per week" but "the eval program is what makes it possible to detect a regression in the safety refusal rate before the model reaches customers; the release blocks the launch when the gate fires; here is the last incident that would have been much worse without it." Concrete, causal, defensible.
- **Where the program sits on the frontier.** The organization's chosen point on the cost-wall-clock-coverage frontier, honestly named. "We are coverage-first on safety and regulator-facing evidence, wall-clock-first on functional-quality gates in CI, and cost-first-adjacent on exploratory research work. The shape reflects our regulatory exposure (EU AI Act high-risk deployment in one product line), our launch cadence (weekly at the flagship product, daily for the developer platform), and our safety commitment (published policy at v2.4)."
- **What the trade-offs are.** The three canonical shapes above, walked briefly, with a clear statement of what would change if the program moved to a different shape. "If we moved to a wall-clock-first shape uniformly, we would drop the safety per-release coverage to a subset and rely on the nightly sweep to catch regressions in the demoted categories; the risk is that a regression there could reach production for up to 24 hours before it is caught."
- **What the ask is.** The specific dollar (and wall-clock and coverage) numbers being requested, ranged, with the assumptions. "We project $X ± 20% in vendor spend for the next fiscal year, up from $Y this year. The delta is driven by expected coverage growth on multi-modal evaluations (the intended-use expansion committed in the product roadmap) plus renewal of the safety-eval red-team contract. Below the ask, here are the reallocation levers we would pull if the constraint tightened."

The narrative is short, cites, and acknowledges. A defense that overstates ("without our eval program the company would fail") loses credibility; a defense that understates ("we mostly run tests") loses funding. A defense that names the trade-off, cites the evidence, and offers reallocation options is the shape leadership can act on.

## Anti-patterns the defense is written against

Four failure modes recur.

- **The line-item defense.** The defense is a spreadsheet of monthly vendor invoices without a narrative. Leadership cannot interpret what the numbers buy. Prevention discipline: the narrative above frames the spreadsheet.
- **The heroic defense.** The defense describes the eval program's achievements as if they were causally necessary for every launch's success. Leadership discounts the claim because it is overstated. Prevention discipline: cite specific incidents where the program did (or would have) caught the regression; do not extrapolate.
- **The "we've always spent this much" defense.** The defense justifies the current budget by pointing at last year's budget. Leadership rightly asks whether the current shape is still the right shape. Prevention discipline: defend the trade-off explicitly, and revisit the shape decision annually.
- **The bottomless-growth defense.** The defense projects continuing growth without ceiling. Leadership rightly asks when the growth stabilizes. Prevention discipline: name the growth drivers (specific product-roadmap items) and their expected duration; a program that promises steady state at year three is more defensible than one that projects indefinite growth.

## Guidance for the defense author

- **Cite the mod-111 warehouse for the cost lines.** Every dollar traces to a per-tenant, per-workload-class row. Untraceable dollars are the failure mode the discipline is against.
- **Name the frontier point.** Coverage-first, wall-clock-first, or cost-first for each part of the program. Do not pretend the program is uniformly optimized.
- **Couple to the launch cadence.** Cost and wall-clock are not independent numbers; they depend on how often the organization ships.
- **List reallocation levers.** Both directions — cut and grow. Leadership will ask.
- **Update annually, at minimum.** The frontier moves. So do the priorities. A locked-in defense that never refreshes will drift into fiction.
- **Compose with Chapter 5.** The build-vs-buy dossier and the budget defense are two views of the same eval-program shape. Keep them coherent.
- **Compose with Chapter 4.** The regulator crosswalk names the coverage the eval program *has to* maintain. Budget cuts that would drop below crosswalk coverage are cuts that would break the compliance chain; the defense should be explicit about which reallocation levers are safe to pull and which are not.

## Summary

The budget defense is the fifth governance artifact this module builds: a leadership-facing narrative about what the eval program spends, on what, and why. Its axes are cost (vendor tokens, compute, storage, human labor, platform-team salary), wall-clock (per-release and full-coverage), and coverage (task, slice, safety-category, regime, depth). The three form a Pareto frontier: any two can be improved at the expense of the third; only substrate improvements shift the frontier itself. Real programs cluster into three shapes — coverage-first for regulated or safety-critical workloads, wall-clock-first for fast-cadence product teams, cost-first for early-stage or under-invested programs — and mature programs are hybrids across the three. The launch cadence couples the axes: cadence sets the wall-clock ceiling and multiplies the cost of coverage. The reallocation levers — shifting coverage to periodic sweeps, moving to cheaper judges, batching, trimming redundant coverage, renegotiating vendor contracts — are what leadership needs to see when the constraint changes. The defense narrative is four paragraphs: what the program buys, where it sits on the frontier, what the trade-offs are, what the ask is. The anti-patterns it is written against — the line-item defense, the heroic defense, the "we've always spent this much" defense, and the bottomless-growth defense — recur when the discipline lapses. Together with the release-gate plan, the model card, the regulator crosswalk, and the build-vs-buy dossier, the budget defense completes the five artifacts this module has built. The exercises turn each artifact into a concrete hands-on deliverable against a chosen product.
