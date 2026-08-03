# exercise-03: Platform Cost Controls

**Estimated effort:** 3 hours

## Objective

Implement the platform-scope cost and parallelism control surface from Chapter 4: per-run and per-tenant token budgets with pessimistic pre-allocation, judge-tier routing driven by task registration, priority queues with reserved concurrency for release-blocking traffic, and rate-limit shaping that keeps the platform under a vendor's declared ceiling. The deliverable is the control surface integrated into the orchestration plane from exercise-02 (or a stub), plus a **load-test demonstration** that shows the controls behaving as advertised under three specific stress scenarios.

## Prerequisites

- mod-111 Chapter 4 (parallelism and cost controls). Chapter 3 (orchestration plane) for the surface the controls attach to; Chapter 5 (warehouse) for the historical distribution the budget projections read from.
- mod-105 for enough context on judge configurations that tier assignment (Tier 0 classical / Tier 1 dedicated / Tier 2 mid-tier frontier / Tier 3 top-tier frontier) is not an abstract label.
- Python 3.11+ with an async framework (`asyncio`, `anyio`, or `trio`), a rate-limit library (`aiolimiter`, `pyrate-limiter`), and access to at least two hosted backends (or one hosted + one mock) so vendor-side variance is visible.

## Requirements

### Part A — token budget accounting

Implement `budgets/token_budget.py` with:

- `TokenBudget(run_id, projected, hard_cap)` — represents a run's budget. Instantiated by the plane at submission time with `projected` drawn from the warehouse's historical distribution (a fallback constant is acceptable for the exercise) and `hard_cap` set to `projected * 1.5` or the caller's explicit budget, whichever is larger.
- `TokenBudget.charge(input, output, cached_input)` — atomically decrements the remaining budget. Raises `BudgetExhausted` if the charge would take the remaining below zero.
- `TokenBudget.remaining()` — returns the remaining budget as a triple.
- `TenantBudget(tenant_id, window, cap)` — rolling-window per-tenant budget. `pre_allocate(projection)` and `release(unused)` calls bracket every run; the tenant's remaining is the cap minus the sum of active pre-allocations plus historical consumption in the window.
- Pessimistic pre-allocation semantics: a run's projected budget is subtracted from the tenant window at submission, and either fully released (if the run under-consumed) or fully consumed (if the run exhausted). Post-hoc accounting is explicitly not implemented.

Ship a unit test that runs a fan-out job of 100 concurrent runs at a projected 50K tokens each against a tenant with a 2M-token daily cap and demonstrates that no more than 40 runs are admitted before the cap blocks the rest.

### Part B — judge-tier routing

Implement `routing/judge_tier.py` with:

- A `JudgeTier` enum with at least four members (`TIER_0_CLASSICAL`, `TIER_1_SMALL_DEDICATED`, `TIER_2_MID_FRONTIER`, `TIER_3_TOP_FRONTIER`).
- Tier registration in the registry (or in a config file for the exercise): each judge revision declares its tier, its per-token cost estimate, and its calibration reference.
- `route_item(task, item, budget_remaining) -> JudgeTier` policy:
  - Default tier comes from the task's registered `default_judge_tier`.
  - Escalation-on-disagreement: if two Tier 1 judges disagree on an item, escalate to Tier 2 (or Tier 3 if the task is release-blocking).
  - Confidence-based escalation: if the tier-N judge emits a confidence below a documented threshold, escalate one tier.
  - Budget-pressure downgrade: if the run's `budget_remaining` falls below a documented fraction of `projected`, downgrade to the next lower tier. Emit an event when the downgrade fires.
- Every routing decision is logged; the warehouse can query "what fraction of this run's items were escalated / downgraded from the default tier."

### Part C — priority queues and reserved concurrency

Implement `queue/priority_queue.py` with:

- Workload classes from Chapter 4: `release-blocking`, `safety-canary`, `research-sweep`, `interactive`, `scheduled-refresh`. Each class has a configurable share of the total concurrency slots.
- Reserved slots: at least 20% of concurrency is reserved for `release-blocking`; if the queue is saturated by lower-priority classes when a `release-blocking` request arrives, the queue preempts a `research-sweep` slot rather than making the release-blocking run wait.
- Preemption semantics: preempted runs are re-queued (they keep their `run_id` and are re-dispatched to the same runner) and their event log records the preemption event with a reason.
- A `QueueMetrics` snapshot API that reports current depth and wait-time percentiles per class.

### Part D — rate-limit shaping

Extend the exercise-02 model-client layer (or ship a standalone `rate_limit.py`) with:

- A token-bucket per backend that is provisioned to 80% of the vendor's declared limit. The bucket refills at a documented rate; requests that would exceed the bucket's current capacity are paced (delayed), not dropped.
- Jittered exponential backoff on 429 / 5xx responses: base 500ms, cap 30s, max 6 retries, jittered by a factor in `[0.5, 1.5]`.
- Backpressure into the queue: when a backend's error rate over the last 5 minutes exceeds a documented threshold (say 20%), the queue slows dispatch to runs targeting that backend by half. Recovery is symmetric.
- Every retry, every pace-delay, and every backpressure state change emits an event that the plane's event sink attributes to the run.

### Part E — the three load-test demonstrations

Write a `LOAD_TESTS.md` that runs three specific stress tests against the plane and reports the observed behavior:

1. **Fan-out under tenant cap.** Submit 100 concurrent runs from one tenant with a small daily cap. Show that pessimistic pre-allocation blocks admissions once the cap is exhausted, that the earlier runs complete, and that budget is released back into the window as they finish. Attach the tenant's budget usage timeline as a plot or table.
2. **Release-blocking preempts research-sweep.** Submit a research-sweep that occupies the queue at 90% concurrency, then submit a release-blocking run 30 seconds later. Show that the release-blocking run starts within a documented preemption deadline (say ≤ 10 seconds) and that the preempted sweep re-queues and eventually completes.
3. **Vendor rate-limit spike.** Artificially inject 429s on one backend for a 60-second window. Show that the rate-limit shaper backs off, that the queue applies backpressure, that no runs fail catastrophically (they either complete via retry, downgrade to a fallback backend if configured, or fail cleanly with a documented error). Attach the backend's error rate and the queue's dispatch rate over the incident window.

## Starter guidance

- **Pre-allocation is easier if the projection is deterministic in the exercise.** Reading from the warehouse for the projection is the real-world approach; a constant per-task projection or a small lookup table works for the exercise and lets you focus on the pre-allocation mechanics.
- **The tenant-budget window is a sliding window, not a calendar day.** A "daily" cap that resets at midnight is easier to implement but produces the midnight-thundering-herd failure mode you want to avoid; a rolling 24-hour window is the intended shape.
- **Escalation-on-disagreement composes with the panel-of-judges pattern.** If the task's default is "run three Tier 1 judges and majority vote," the escalation triggers when the majority is thin (2/3 rather than 3/3). Chapter 4 sketched this; the exercise can implement a simplified version.
- **Preemption of a running run is the trickiest bit.** For the exercise, "preempt" can mean "signal the adapter to stop after the current in-flight item" rather than a hard interruption; the adapter re-queues the remaining items in a follow-up run with the same `run_id` parent.
- **Do not skip the events.** Every control-surface decision produces an event. The load tests read the events to verify behavior; without them, all you can inspect is aggregate numbers.

## Acceptance criteria

- Token budgets pre-allocate on submission and release on completion; a fan-out job cannot silently blow through a tenant cap.
- Judge-tier routing is driven by task registration, respects escalation and confidence rules, and downgrades under budget pressure. Every routing decision is queryable.
- The priority queue enforces reserved concurrency for `release-blocking` and preempts a `research-sweep` when necessary; preemption re-queues cleanly.
- The rate-limit shaper stays under 80% of the vendor's declared limit under normal load and backs off exponentially under 429 pressure. Backpressure propagates into the queue.
- `LOAD_TESTS.md` shows all three demonstrations running end-to-end with the observed behavior matching the acceptance criteria; deviations are documented with a hypothesis.
- Every control-surface action emits an event that names the run, the actor, and the reason.
- The plane never OOMs, deadlocks, or silently drops runs under any of the three stress scenarios.

## Stretch goals

- **Prompt caching integration.** Wire the model client's judge calls to opt into vendor prompt caching (Anthropic's `cache_control`, OpenAI's automatic caching, Google context caching). Report the observed cache-hit rate and the token-cost delta on a judge-heavy run. This is the single highest-leverage optimization in the chapter.
- **Batch API integration.** For `research-sweep` and `scheduled-refresh` classes, ship a code path that submits eligible items through a vendor batch API (Anthropic Message Batches or OpenAI Batch) instead of the synchronous endpoint. Demonstrate the ~50% cost reduction on a synthetic 500-item run.
- **Multi-backend fallback.** When a backend hits sustained rate-limit backpressure, route eligible runs to a fallback backend of the same tier (both Anthropic Sonnet-class or both OpenAI 4o-class). The fallback is declared at task-registration time; a task without a fallback declaration is not eligible for cross-backend routing.
- **Cost attribution query.** Implement Chapter 5's cost attribution query (per-team, per-workload-class, over a quarter) on top of the events emitted by the control surface. This is a hand-off to exercise-04.
