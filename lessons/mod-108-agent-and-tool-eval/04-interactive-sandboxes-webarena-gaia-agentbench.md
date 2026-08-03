# Interactive Sandboxes: WebArena, GAIA, and AgentBench

SWE-bench's environment is inert: a repository at a commit, dependencies installed, tests waiting to be run. The agent's actions read and edit files, and the "world" changes only in ways bounded by the container's filesystem. Interactive-sandbox benchmarks are structurally harder. WebArena is a live web application the agent clicks through. GAIA gives the agent a browser and a set of open-domain research questions that require multi-hop reading and reasoning. AgentBench spans eight environments, each with its own state model, from a real Ubuntu terminal to a MySQL database to a KG-search interface. In every case, the environment mutates as the agent acts on it, and *keeping the environment reset, versioned, and deterministic* is where most of the engineering effort in these benchmarks goes. This chapter covers the design pattern the three benchmarks share, and the sandbox-isolation and deterministic-replay considerations that decide whether your run is reproducible.

## What "interactive sandbox" means

A benchmark is interactive-sandbox-shaped when:

- The agent's tools produce *side effects* on a stateful environment (clicking a Buy button, deleting a file, running a shell command).
- The success criterion is evaluated against the *state of the environment* after the trajectory, not (only) against the text the agent emitted.
- The environment must be *reset* between samples so that a modification in sample `i` does not contaminate sample `i+1`.

WebArena (Zhou et al. 2024, *WebArena: A Realistic Web Environment for Building Autonomous Agents*, ICLR 2024) and its successor VisualWebArena (Koh et al. 2024, ACL) are the canonical example. The agent operates a browser against locally-hosted copies of five web applications — OneStopShop (an e-commerce site), Reddit (a social forum), GitLab (a code-hosting site), a CMS (Magento), and OpenStreetMap. Each task ("find the cheapest red shirt on OneStopShop and add it to your cart") is graded by a Python function that inspects the browser state and the underlying application database, not the assistant's text.

GAIA (Mialon et al. 2024, *GAIA: a benchmark for General AI Assistants*, ICLR 2024) is a research-benchmark shape. The agent is given a browser and general internet access, plus a set of tools (file reader, calculator, web search), and asked open-domain questions ("How many albums did the artist X release before 1990?") that require multi-hop research. Success is exact-match against a canonical short answer, but the environment (the live internet) is not reset — you cannot reset the web — so *replay determinism* is the operative concern rather than *state isolation*.

AgentBench (Liu et al. 2024, *AgentBench: Evaluating LLMs as Agents*, ICLR 2024) is a portfolio: eight environments (OS shell, database, knowledge-graph, digital card game, house-holding, web shopping, web browsing, lateral-thinking puzzles). Each environment has its own state model, its own success criterion, and its own reset story. AgentBench is useful precisely because it exposes the *variety* of environment shapes — a scorer that works on WebArena needs adaptation for AgentBench-OS and doesn't apply to AgentBench-KG at all.

The chapter walks the two big design axes — **sandbox isolation** and **deterministic replay** — using WebArena and GAIA as the concrete examples, and returns to AgentBench for the portfolio question at the end.

## Sandbox isolation: what has to be reset per sample

The isolation properties from mod-107 Chapter 2 (process, filesystem, network, memory) all still apply to the container the agent's tools run in. Interactive-sandbox benchmarks add three more layers:

### 1. Application state

The web applications WebArena wraps are ordinary Rails / PHP / Python services backed by a database. Between samples the database must be reset to a known-good state — otherwise a task that says "add the red shirt to the cart" starts sample `i+1` with sample `i`'s cart still full. WebArena's official Docker Compose stack ships a `reset_state.sh` script that restores the database from a fixture dump; the harness invokes it between samples.

Two ways this quietly goes wrong:

- **Partial reset.** The database is reset but the Redis session cache, the Elasticsearch index, or the filesystem-attached user uploads are not. Sample `i+1` sees stale sessions or stale search results. Fix: reset every stateful component, or (much better) tear down the whole compose stack and bring up a fresh one per sample. WebArena's reference reset is fast enough that per-sample teardown is feasible for small runs; for larger runs the harness batches resets across parallel workers.
- **Reset-then-warm cache.** After reset, the first request against a service hits a cold cache and takes 10× normal latency. If your tool has a per-request timeout, cold-cache requests time out and the agent sees flaky tools. Fix: warm the caches after every reset with a scripted set of dummy requests.

### 2. Time and randomness

Applications embed the current date in output (Reddit's "posted 2 hours ago", OneStopShop's default shipping ETA). If a task's success criterion depends on a date, running the same task a month later can flip the answer. The correct posture is *frozen time* — mock the system clock inside the containers, or configure the applications to use a fixed reference date. WebArena's fixtures include a reference `NOW` for applications that need it; verify per sample that the frozen clock is active before scoring.

Randomness enters similarly. If an application uses random sampling anywhere (pagination order, recommendation rank), fix the seed at reset. AgentBench's card-game environment is the extreme case — a fresh shuffle per sample defeats reproducibility entirely; the reference implementation seeds the shuffle from the sample ID.

### 3. Network egress

WebArena is explicitly *offline* — the applications are localhost services, and the containers are network-isolated so the agent cannot escape to the live internet. This is a deliberate design choice: keeping the network closed means the benchmark is deterministic and safe to run in shared environments. Verify egress isolation at run start with a probe (the agent tries to `curl example.com`; the tool result should be a connection refused, not a 200); a WebArena instance where the network isolation slipped is a WebArena instance whose scores are contaminated by whatever the live internet returned that day.

GAIA is the opposite — its whole design is that the agent has open internet access, because the tasks require researching real websites. This makes GAIA fundamentally *not deterministic*: the same task on the same date can return different answers depending on whether a Wikipedia article was edited, a page moved, or a search engine reranked. GAIA's reproduction strategy is not "reset the environment" (the environment is the whole internet) but "record traffic and replay" — see the next section.

## Deterministic replay: the four strategies

The determinism problem is: your agent visits a site today, the site changes tomorrow, and now the trajectory does not reproduce. Four strategies, in decreasing order of fidelity:

### Strategy 1 — Contain the environment (WebArena)

The environment is a bundle of Docker containers you host locally. Nothing external. Determinism is a property of "same Compose stack, same fixtures, same seed." This is the strongest form of determinism and it is the reason WebArena is the right first stop for interactive-sandbox eval. The cost: you can only evaluate the model against the specific web applications in the bundle. WebArena's five are a defensible sample of the web-agent design space but they are not the live internet.

### Strategy 2 — Snapshot and replay (WARC / mitmproxy)

The agent hits live sites during a *record* run; a HTTP proxy records every request/response into a WARC (Web ARChive) file. Subsequent *replay* runs route the agent's HTTP through the same proxy, which serves the recorded response. mitmproxy, Wayback Machine's `wayback-recorder`, and the Playwright Network Replay pattern are the common tools. Replay works for GET-heavy workflows; POST workflows that mutate site state cannot be replayed faithfully because the server's response depends on real writes.

### Strategy 3 — Freeze a snapshot of the source (Common Crawl / Wayback URLs)

Instead of live URLs, tasks are re-anchored on `https://web.archive.org/web/<timestamp>/<url>` — every question refers to a fixed snapshot in a public archive. Reproducibility becomes a property of "Wayback returned the same bytes at the same URL" — a weaker guarantee than a local record (Wayback occasionally has intermittent failures and does not archive every asset), but it is the default fallback for open-web tasks where you cannot host a proxy.

### Strategy 4 — Report and accept non-determinism (GAIA, as run)

GAIA's official evaluation server accepts submissions and scores them against a hidden answer key; the leaderboard reports a single accuracy number. The determinism cost is passed to the submitter: two runs of your agent against GAIA on different days can produce different trajectories and different accuracies. The strategy is to run multiple epochs, report `pass^k` alongside `pass@k`, and note the run window in the manifest. This is the least reproducible of the four strategies and is only defensible because GAIA's tasks are open-domain research where any other strategy sacrifices the benchmark's whole point.

The chapter's operational point: **pick the strategy that matches the benchmark's design and pin it in the manifest**. Do not run WebArena tasks against the live e-commerce sites they were adapted from; do not run GAIA against a stale WARC and pretend the internet did not move.

## The success-criterion function

Interactive-sandbox benchmarks depend on an executable success-criterion function that inspects the *environment* after the trajectory, not the assistant's final text. WebArena's ships an evaluator per task template — a Python callable that queries the target application's database or introspects the browser DOM and returns `True` / `False`. Three categories:

- **URL-and-DOM checks.** Did the agent end on the expected URL, and does the DOM contain the expected element? Used for "navigate to page X and find element Y" tasks.
- **Database-state checks.** Query the application's underlying DB. Used for "add item to cart" (`SELECT * FROM cart WHERE user_id = ...`), "post a comment" (`SELECT text FROM comments WHERE ...`), and any task with a persistent effect.
- **Answer-string checks.** For QA-style tasks where the agent's final answer message is what matters, an exact-match or fuzzy-match against a target string. GAIA uses this exclusively; WebArena uses it as a fallback for some read-only tasks.

The evaluator is the *most-scrutinized* line of code in an interactive-sandbox benchmark, because it is where under-specification and over-specification of the task collide. Two failure modes:

- **Over-specified evaluator.** The task says "add the red shirt to your cart"; the evaluator checks for a specific SKU. The agent added a differently-SKU'd red shirt (the site had two); the evaluator returns False even though the agent solved the task-as-stated. This shows up as a systematic under-report of the model's capability on ambiguous tasks.
- **Under-specified evaluator.** The task says "delete the top-voted comment on post X"; the evaluator only checks that *any* comment was deleted on post X. The agent deleted a random comment; the evaluator returns True. This shows up as an over-report — the agent looks capable but is not solving the task.

Neither is fixable purely inside the agent-eval harness. Both are benchmark-authorship bugs (mod-102). The pragmatic harness posture: log the evaluator's decision reasoning per sample (which DB row it queried, which DOM element it looked for, which regex it applied) so a reviewer can audit systematic evaluator failure modes. A benchmark that ships evaluators as opaque functions and does not surface their reasoning is a benchmark whose numbers you should treat as approximate.

## Partial credit on trajectories

Most interactive-sandbox benchmarks are binary at the task level — the agent either solved it or did not. This is a coarse instrument. A trajectory that reached the checkout page and added the wrong item is closer to success than one that got confused and never left the login page; a binary scorer collapses both to failure and hides the diagnostic. Chapter 5 (the LLM-judge chapter of this module treats partial-credit rubrics for free-form trajectories as its own topic, but two lightweight partial-credit constructs are worth mentioning in the sandbox context:

- **Milestone predicates.** For each task, decompose the success criterion into 2–5 ordered milestones (`logged in`, `navigated to product`, `product added to cart`, `checkout initiated`). The evaluator returns a fraction: milestones-reached / milestones-total. This is easy to author for structured tasks; WebArena's fork `WebLINX` explicitly reports milestone-fractional scores.
- **Progress-at-cap.** Report the *distance to success* when the agent hits its budget. For a shopping task, "cart contains 1 of 2 required items" is a scorable state even without a milestone decomposition; a per-task judge (Chapter 5) can produce a `0.0 – 1.0` progress score post-hoc from the trajectory and the final environment state.

Both are *reporting* additions, not replacements for the binary metric. The binary metric is what leaderboards compare on; partial-credit signals are what you show internally to debug why your agent scored 0 on 40% of tasks.

## Cost and latency on interactive benchmarks

Where SWE-bench trajectories are dominated by *inference cost* (long completions), interactive-sandbox trajectories are often dominated by *tool latency*: a page load is 1–3 seconds, a form submission triggers a re-render, an OS command takes wall-clock time to execute. Total wall-clock per WebArena trajectory can exceed 5 minutes even when the total prompt+completion tokens are modest. Two reporting habits:

- **Report `wall-clock per trajectory` alongside `steps per trajectory`.** Two agents with the same step count can have very different wall-clocks depending on whether they parallelize tool calls, whether they pipeline actions, and whether they wait for page loads properly.
- **Report `retries per trajectory`.** Interactive tools flake more than SWE-bench's `pytest`. If the trajectory shows a lot of retries, the browser or the app is unstable, and that instability inflates the wall-clock and confuses the model. High retry counts on a specific task cluster is a benchmark-side issue to escalate, not a model-side signal.

## AgentBench's portfolio pattern

AgentBench is a portfolio of eight environments; each is a mini-benchmark with its own evaluator, its own state model, and its own reset story. The value of AgentBench is *not* the individual environments (each has a better dedicated benchmark elsewhere — WebArena for web, HumanEval-scale sandboxes for OS-shell) but the *comparability across environments* on the same model. A model's per-environment score profile is diagnostic: strong on OS-shell but weak on KG-search suggests the tool-calling capability is fine but the multi-hop reasoning is not; strong on KG-search but weak on web-browsing suggests the reverse.

For a serious agent-eval suite, AgentBench is a useful *cross-cutting comparability probe* — a lightweight run to see whether your model behaves similarly across environment shapes — but not a substitute for a deeper run on the environment your product actually cares about.

## Where these benchmarks work and don't

**Work well for.** Comparing web-agent implementations on WebArena. Comparing multi-hop research capability on GAIA. Getting a coarse cross-environment capability profile on AgentBench.

**Do not work well for.** Reproducing a published number without the same Compose stack (WebArena) or the same run window (GAIA). Comparing agent implementations on tasks materially different from the benchmark's fixed set. Measuring product-specific web-agent quality — a WebArena number is a number on WebArena's five sites, not on your actual product surface.

## Summary

Interactive-sandbox benchmarks (WebArena, GAIA, AgentBench) add three layers of statefulness on top of the mod-107 sandbox: application state that must be reset between samples, time and randomness that must be frozen or seeded, and network egress that must be closed (WebArena) or explicitly managed via record-and-replay (GAIA). Determinism is a design choice with four strategies (contain, snapshot-replay, frozen-source, or accept-and-report); the strategy must match the benchmark and be pinned in the reproducibility manifest. The success-criterion function inspects environment state, not text, and its own failure modes (over- or under-specification) are benchmark-authorship bugs that the harness cannot fix but can surface via reasoning logs. Report per-trajectory wall-clock and retry counts alongside step counts; report milestone-fractional progress as a partial-credit signal alongside the binary leaderboard metric. AgentBench's contribution is cross-environment comparability, not depth on any single environment. Chapter 5 turns to the partial-credit rubric — the LLM-judge-of-trajectories instrument that lets you score free-form agent runs where no clean environment-state predicate is available.
