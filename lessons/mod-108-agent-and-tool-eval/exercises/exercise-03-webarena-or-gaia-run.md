# exercise-03: WebArena or GAIA Run

**Estimated effort:** 5 hours

## Objective

Run a task subset from an interactive-sandbox benchmark — **WebArena** (or VisualWebArena) or **GAIA** — against a candidate agent, produce the headline pass rate for the subset, and — the load-bearing part of this exercise — **explicitly reason about sandbox isolation and deterministic replay**. The deliverable is not just a number; it is a written argument for whether your number is *reproducible*, defended by evidence from environment probes, run logs, and the reproducibility manifest. Chapter 4's four determinism strategies are the frame; the exercise is picking one and defending the pick.

## Prerequisites

- mod-108 Chapter 4 (interactive-sandbox benchmarks).
- Exercise 01's trajectory scorer (you will run it against your agent's traces).
- Docker + Docker Compose. WebArena's full stack needs **~24 GB RAM headroom** for the five application containers plus the browser sandbox; GAIA needs no local sandbox but does need a network-permitted environment. If you cannot allocate the memory for WebArena, choose GAIA.
- Access to a hosted LLM. Budget: WebArena at 30–50 tasks against a modern chat model with a bounded step budget is typically **$20–$100**. GAIA is cheaper per task (~$0.20–$2 per task) but validation-set questions can require many search rounds; budget **$40–$150** for a meaningful subset.

## Pick one: WebArena or GAIA

The two benchmarks answer different determinism questions, so pick the one whose determinism story matches what you want to defend:

- **WebArena.** Fully-contained sandbox (five local Dockerized web apps). Determinism is a property of "same Compose stack, same fixtures, same reset script." Task success is graded by executable predicates on browser + DB state. Pick this if you want to defend Chapter 4 **Strategy 1 (contain the environment)**.
- **GAIA.** Open-web research questions. The environment is the live internet. Determinism is degraded and the strategy is either **record-and-replay** (Strategy 2) or **accept and report non-determinism** (Strategy 4). Pick this if you want to defend one of those two.

State your pick at the top of the report with one paragraph on why.

## The subset

You do not have to run the full benchmark. A **subset of 30–60 tasks** is enough to demonstrate the pipeline and produce a pass-rate with a meaningful CI. Pick the subset with a rule that a reader can reproduce (not "the ones that were cheap"):

- WebArena: task IDs 0–59, or the first 10 tasks from each of the five sites (approximately balanced across applications).
- GAIA: the first 30 items from the *validation* set (not the test set — the test set is graded server-side and does not give you per-item labels).

Document the subset in the report.

## Requirements

### Part A — stand up the sandbox

**WebArena route.** Clone the WebArena repo, launch the reference Docker Compose stack, and confirm all five web applications are reachable at their expected localhost URLs. Run the reset script; confirm that a browser navigation to (e.g.) OneStopShop's homepage returns the *same* content bytes after two consecutive resets (verify by hashing the response body — bit-for-bit, or at least header-set equality). This is your Strategy 1 evidence.

Then run three probes and log the outcomes:

1. **Egress probe.** Have your agent tool `curl example.com` from inside the sandbox. Expected: connection refused / DNS failure. If it returns a 200, your network isolation is broken.
2. **State-reset probe.** Add an item to OneStopShop's cart via a scripted action, then invoke reset, then query the cart. Expected: empty. If non-empty, your reset is partial.
3. **Frozen-clock probe.** Query Reddit for a page that renders a "posted N days ago" timestamp; run the query on two different real-world dates. Expected: same rendered text. If it differs, your frozen-clock is off (WebArena's applications should be running against a fixed reference date; verify the fixture).

**GAIA route.** No local sandbox. Confirm that the tools available to your agent — web search (Bing / Google / DuckDuckGo / SerpAPI), file reader, calculator, Python execution — are reachable. Pick your replay strategy:

- **Strategy 2 (record-and-replay).** Set up mitmproxy or an equivalent to record all HTTP traffic during the run. Ship the resulting `.mitm` or WARC bundle as an artefact. Later runs can be replayed against the bundle.
- **Strategy 4 (accept and report).** Do not record; report the run as non-deterministic and document the run window (start / end UTC timestamps) in the manifest. Run 2–3 epochs and report per-epoch variance.

State your choice explicitly.

### Part B — run the agent

Pick an agent implementation:

- **WebArena reference agent.** WebArena ships a reference agent (`BrowserEnv` with a small solver). Use it if the goal is comparability with published numbers.
- **Your own agent** built on top of a Playwright-backed browser tool set (WebArena) or a search + file + Python tool set (GAIA). Correct for internal comparisons; will not match public numbers without careful prompt tuning.
- **A published agent** (BrowserGym, WebArena's baselines, an OpenHands run configured for browsing). Cite the version.

Set a documented budget: `max_actions` for WebArena (30–50 is typical for the subset), `max_turns` for GAIA (20 is typical). Log per-task trajectories in a format that adapts into your exercise-01 canonical schema.

### Part C — score

For WebArena, the harness's built-in evaluator returns per-task pass/fail plus (for many task templates) a reasoning trace of which DB row it checked. Log the evaluator's reasoning per task.

For GAIA, compare each final answer to the target via the GAIA-recommended normalizer (lowercase, strip punctuation, number normalization). Log both raw and normalized final answers.

Compute:

- **Pass rate** with a 95% CI (bootstrap over tasks).
- **Per-category breakdown.** WebArena tasks are tagged by site (OneStopShop, Reddit, GitLab, CMS, OpenStreetMap) and by task template; GAIA validation tasks are tagged by "level" (1, 2, 3 by difficulty). Report per-slice.
- **Termination reasons.** `submitted`, `budget_exhausted`, `harness_error`, distribution.
- **Wall-clock and step distributions**, median and p95.

### Part D — the trajectory-scorer signals

Adapt your agent's per-task trace into the exercise-01 canonical schema and score. Report:

- Tool-call validity rate. (For WebArena, "did the browser action's arguments validate against the DOM?" is the analogue; if you can only compute JSON-shape validity, note it.)
- Tool-execution success rate. WebArena tools frequently return `error` when a click misses; the rate is expected to be lower than SWE-bench.
- Redundant-call rate. Interactive-sandbox agents often re-issue the same query when they don't parse a result; a high redundant-call rate is a diagnostic.
- Mean tool calls per trajectory, per cohort (pass / fail / budget_exhausted).

### Part E — the determinism argument

Ship `docs/determinism.md` (1–2 pages) covering:

1. **Chosen strategy.** Which of the four Chapter 4 strategies you picked, and why.
2. **Evidence.** Probes from Part A with their outcomes. For WebArena, the egress + state-reset + frozen-clock results. For GAIA record-and-replay, a demonstration that a replay run reproduces the recorded trajectory. For GAIA accept-and-report, the per-epoch variance.
3. **Known gaps.** Every place your determinism strategy is incomplete. Examples: "state reset does not clear the Redis session store; sessions from previous runs may leak;" "record-and-replay does not cover POST requests that mutate remote state;" "run windows overlap known Wikipedia edit events that changed the answer for question X."
4. **Manifest pointers.** How the manifest reflects the choice — Compose file SHA (WebArena) or WARC bundle hash (GAIA-record) or run-window timestamps (GAIA-accept).

### Part F — the report

Ship `REPORT.md` (2–3 pages) with:

1. Setup (benchmark, subset, model, agent, budget).
2. Pass rate with CIs, per-slice breakdown, termination-reason distribution.
3. Trajectory-signal summary from Part D.
4. Cost distribution.
5. Determinism argument summary (Part E is the appendix).
6. Reproducibility manifest.

## Starter guidance

- **Run the sandbox probes before running the agent.** For WebArena, a broken reset script or a leaking network is a run-killer. Discover it in Part A, not after $80 of agent inference has produced numbers you cannot defend.
- **Frozen clock matters more than you'd think.** Several WebArena tasks are date-sensitive ("show me the top posts from the last week"). If the applications are using the host clock, running the same subset today and next month gives you different task outcomes even at 100% agent skill. The Compose stack has explicit clock-freezing config; use it.
- **On GAIA, log your search results.** The single largest source of non-determinism on GAIA is that search engines rerank. If you log the raw search-result payloads, a reader can see whether your agent got "unlucky" or "lucky" search results relative to a replay run.
- **Do not run WebArena and GAIA both.** Pick one. The exercise is about making the determinism argument end-to-end, not about surface coverage.
- **Cite the WebArena / GAIA version.** WebArena's Compose stack has been re-released; GAIA's evaluation server URL has moved. The manifest must pin the commit SHA of the benchmark repo and the version of the Docker images.
- **Report subset in the numerator.** "Pass rate = 34% on 40 tasks" is a claim; "pass rate = 34%" without the subset size is not. Attach a `tasks_run.txt` listing exact task IDs.
- **A low pass rate is fine.** Modern agents on WebArena's full test set score in the 20–40% range; on GAIA's Level 3 validation, in the single digits. If your subset returns 12% pass rate, that's a real number, not a broken run — as long as the probes in Part A pass.

## Acceptance criteria

- The chosen sandbox / determinism strategy is stated explicitly and defended with the Part A probes (WebArena) or the Part A replay demonstration (GAIA record-and-replay) or the per-epoch variance report (GAIA accept-and-report).
- Pass rate with a bootstrap 95% CI, per-slice breakdown, and termination-reason distribution are reported for a documented subset of 30–60 tasks.
- Trajectory-signal report (Part D) covers validity, execution success, redundancy, and per-cohort mean tool calls.
- Reproducibility manifest pins model, agent, benchmark version (repo SHA + Docker digests or WARC hash), budget, and run window.
- The determinism-gaps section in `docs/determinism.md` names at least three specific places your strategy is incomplete.

## Stretch goals

- **Reset-cost measurement.** For WebArena, time the reset step. Report mean and p95 reset wall-clock. Reset is a hidden per-sample cost; a benchmark that resets in 30s per sample times 500 samples is > 4 hours of pure reset overhead.
- **Multi-epoch variance on GAIA.** Run 3 epochs against the same subset. Compute per-question pass consistency: fraction of questions that pass on all epochs, on some, on none. This is your empirical measure of GAIA's non-determinism — expect the "unstable" bucket to be non-trivial.
- **Milestone rubric.** For 10 WebArena tasks, hand-author milestone predicates (Chapter 5 Shape 1) and score. Report the mean milestone fraction alongside the binary pass rate. Milestones catch "the agent reached checkout but picked the wrong item" trajectories that binary pass misses.
- **Cross-model.** Run the same subset against two models on the same agent + budget. Report the pass-rate gap and the trajectory-signal gap. Argue whether the model with higher pass rate is *better* or just *lucky* on this subset — the CIs will often overlap.
- **Adversarial reset probe.** Before running the agent, deliberately break the reset script (leave a decoy item in a cart, seed a stale session) and confirm the harness detects and refuses to run. If it does not, the sandbox is missing a defense-in-depth check that a real leaderboard operator needs.
