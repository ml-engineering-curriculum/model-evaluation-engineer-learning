# exercise-05: Eval in CI with SLOs

**Estimated effort:** 4 hours

## Objective

Wire the eval platform from exercises 01–04 into a CI/release pipeline at the three altitudes Chapter 6 names — pull-request smoke tests, release-candidate promotion (the actual gate), and post-deploy scheduled runs — and stand up the platform's own SLO discipline (availability, latency, correctness, cost predictability) with targets derived from the warehouse's own historical distribution. The deliverable is a working end-to-end pipeline in which the CI system *submits through the platform* rather than executing evals itself, an override path that lands in the warehouse as a first-class row, and an SLO dashboard whose targets are queries against the warehouse, not guesses.

## Prerequisites

- mod-111 Chapter 6 (CI, release pipelines, and the platform's own SLOs). Chapters 2–5 for the registry, orchestration plane, cost controls, and warehouse the CI layer sits on top of; the exercises 01–04 outputs (or the stubs called out below) are the substrate the pipeline drives.
- mod-110 Chapter 2 (offline regression gates) — enough familiarity with the `regsuite`-style gate DSL that "verdict is opaque to the pipeline" is not an abstract prescription.
- Google SRE Book Chapter 4 (Service Level Objectives) and Chapter 3 of *The SRE Workbook* — the reference for the error-budget mechanic this exercise borrows.
- A working CI substrate: GitHub Actions, GitLab CI, Buildkite, Argo Workflows, or a local `act`/`makefile`-driven equivalent. The point of the exercise is the integration shape, not the CI tool.
- Python 3.11+ with `requests` (or `httpx`), `pyyaml`, `sqlalchemy`, plus whatever your CI substrate expects. A local Prometheus + Grafana (or an equivalent metrics stack) is useful for Part D but not required if you emit metrics to a queryable table in the warehouse instead.

## Substrate assumptions

The exercise assumes the outputs of exercises 01–04 are available. If you are starting fresh, ship a minimal stub for each:

- **Registry (from exercise-01):** a table that resolves `(kind, name, semver)` to a content hash. A dozen rows is enough.
- **Orchestration plane (from exercise-02):** a `POST /v1/eval-runs` endpoint that accepts a request and returns a `run_id`, plus a `GET /v1/eval-runs/{run_id}` that returns status and (once complete) the resolved artifact bundle. A synchronous in-process runner is acceptable; the point of this exercise is not the plane's async correctness.
- **Cost controls (from exercise-03):** workload-class-aware admission. `release-blocking` runs get reserved concurrency; `research-sweep` runs can be preempted.
- **Warehouse (from exercise-04):** the `eval_run`, `run_aggregate_metric`, `run_event_log`, `run_cost` tables, populated by the plane's event stream.

Document in `SUBSTRATE.md` which components you built for real and which you stubbed, so a reader can tell what the SLO numbers actually depend on.

## Requirements

### Part A — pull-request smoke tests (fast, cheap, fail-closed on schema)

Ship a CI job that runs on every pull request to the eval-config repo (or a subdirectory of the model repo that hosts registry additions). Wire it to your CI substrate as a required check.

Contents:

- **Schema validation for registry additions.** The job runs `registry rebaseline-check` (from exercise-01) on any added or modified artifact revision. A PR that registers a judge with an unversioned backend model fails; a PR whose declared semver bump disagrees with the diff classification fails.
- **A short sanity-run on a fixture dataset.** If the PR touches a task, the job submits an eval-run to the plane against a 5–20-item fixture dataset (registered separately as `<task>-fixture@rev`), pinned to a **Tier 0 or Tier 1** judge only. Budget: single-digit-dollars vendor spend, wall-clock target under 5 minutes.
- **Diff-aware re-baseline warning.** If the PR touches a judge whose registered consumers include any active release-blocking gate, the job posts a PR comment naming the affected gates and requires an explicit `re-baseline-ack` reviewer approval before the PR is mergeable.

The job **fails closed** on schema errors (a schema error is a broken registration and merging it silently breaks the registry). The job **fails open** on smoke-run flakiness (a transient vendor 500 during a fixture sanity-run is not a merge-blocker; the fix is a re-run button, not a policy exception). Document the distinction in `CI.md` so a reader can tell which failures are load-bearing.

### Part B — release-candidate promotion (the actual gate)

Ship a `release-candidate-eval.sh` (or equivalent pipeline job) that a release pipeline invokes when a candidate model is promoted from the training system into a release-candidate state.

The invocation flow, implemented literally:

1. The pipeline resolves the suite reference (`suite_id = "release-regression@rev-N"`) through the registry and records the resolved hash in a local `release-audit.jsonl`. The pipeline **refuses** to accept a `suite_id` that has been mutated in place (i.e., a hash that has changed since the pipeline last saw it) without an explicit `--accept-suite-change` flag documented in the audit trail.
2. The pipeline calls `platform.submit_suite(suite_id=..., model_ref="candidate:sha256:...", priority="release-blocking", budget_tokens=..., attribution={"team": "...", "release_id": "..."})` and receives a `suite_run_id`.
3. The pipeline polls `suite_run_id` (or subscribes to a webhook if your plane supports one) until the run reaches a terminal state.
4. The pipeline retrieves `verdict = platform.get_verdict(suite_run_id)` — a small JSON of `{"status": "PASS|WARN|FAIL", "blocking_failures": [...], "warnings": [...], "report_url": "..."}` — and gates on `verdict.status`.

Three properties are load-bearing and must be verifiable from the pipeline's own code:

- **The pipeline reads `verdict.status` and nothing else.** The pipeline does not `if verdict.metrics["helpfulness"] < 0.7:` — that logic lives in the gate DSL, in the registry, versioned. A reader auditing the pipeline should see one line of gate logic (`if verdict.status == "FAIL": sys.exit(1)`).
- **The pipeline records the resolved suite hash in `release-audit.jsonl` before submitting.** A silently-swapped suite would land a different set of gates without provenance; the audit record is what makes the swap visible after the fact.
- **The pipeline records the `suite_run_id`, the `verdict`, and the terminal warehouse row IDs in `release-audit.jsonl`.** Every release has a paper trail that resolves through the warehouse.

Author a `release-audit.jsonl` example from a synthetic pass and a synthetic fail so a reader can see the shape without running the pipeline.

### Part C — override as a first-class API

Ship the override path. This is the seam where the release manager's "we know about the failure, we're shipping anyway" becomes durable state instead of an out-of-band Slack message.

- **`POST /v1/verdicts/{suite_run_id}/override`** on the plane, accepting `{"actor": "...", "reason": "...", "override_id": "OVR-YYYY-NNNN", "expires_at": "..."}`. The endpoint requires an authenticated actor with the `release-override` role (a config file lookup is acceptable for the exercise) and writes an `override` row to the warehouse with a foreign key to the `suite_run_id`.
- **The pipeline consults the override before failing.** On `verdict.status == "FAIL"`, the pipeline calls `platform.get_active_overrides(suite_run_id)`. If a non-expired override exists, the pipeline continues but records `override_applied: true` in the audit line and posts a comment on the release ticket naming the override actor, reason, and expiry.
- **Overrides expire.** No perpetual overrides. The exercise picks a default (say 14 days) that is enforced at read time, so an expired override does not silently keep a broken gate open.
- **Overrides are auditable.** Add a `warehouse/queries.py` function `overrides_in_period(start, end)` that returns every override applied in the window with the actor, reason, expiry, and the `run_aggregate_metric` values the override waved through. This is the query the safety review / release retro consumes.

Write `OVERRIDE.md` explaining the process for a release manager. Two paragraphs is enough — the process is what makes the mechanism useful; a mechanism without a documented process is a bypass with extra steps.

### Part D — the four platform SLOs

Ship `slo/definitions.yaml` that declares the platform's SLOs with targets derived from the warehouse, not chosen a priori. For each SLO, define: name, workload-class scope, measurement window, target, error budget, alert threshold, alert routing.

Required SLOs:

- **Availability (per workload class).** The percentage of eval-run requests over the window that reach a terminal state (`succeeded`, `failed`, `cancelled`, `budget_exceeded`) without a `platform_error`. Targets: 99.0% monthly for `release-blocking`; 99.5% monthly for `interactive`; 99.0% weekly for `scheduled-refresh`. Platform errors are things the platform is at fault for (queue crashes, worker OOMs, ingestion failures); vendor 429s that eventually succeed on retry are *not* platform errors.
- **Latency (per workload class).** Wall-clock time from `submitted_at` to `completed_at`, P95. Targets are derived from the warehouse's rolling 90-day distribution (see the query in Chapter 6). Ship a `slo/derive_targets.py` script that reads the warehouse, computes P95 per workload class over the last 90 days, and proposes a target that is `P95 * 1.1` (a 10% headroom). The proposed targets are written back into `slo/definitions.yaml` as `# derived: ...` comments for a human to promote to the `target:` field.
- **Correctness.** Two sub-metrics: **reproducibility** (percentage of completed runs whose `platform.rerun(run_id)` produces byte-identical aggregate values, target 99.9%) and **verdict fidelity** (percentage of `release-blocking` runs whose `verdict.status` was consumed as-declared by the pipeline — pass proceeds, fail blocks or has a recorded override, target 100%).
- **Cost predictability.** Two sub-metrics: **in-budget completion rate** (percentage of runs that complete within `budget_tokens`, target 95%) and **platform-wide budget adherence** (monthly aggregate spend within the declared platform budget, target 100% within a ±10% tolerance).

Ship `slo/report.py` that runs the SLO queries against the warehouse and emits a Markdown report (`slo/report-YYYY-MM.md`) with adherence numbers per SLO and a color-coded PASS/WARN/BURN status.

### Part E — the eval-in-CI failure-mode demonstrations

Write `FAILURE_MODES.md` that demonstrates the two Chapter 6 anti-patterns are caught by your setup.

1. **Eval runs inside the CI job's own process.** Write a `bad-ci.yml` (or equivalent) that runs `python run_evals.py --model=$CANDIDATE` directly in the CI runner and pipes stdout to a `pass/fail` grep. Compare it side-by-side with the release-candidate pipeline from Part B. Enumerate the specific consequences from Chapter 6 (CI runner budget, no warehouse row, silent Python-env drift, no cost attribution) and show which of your Part B mechanisms addresses each.
2. **The green-light gate that never turns red.** Modify the plane to unconditionally return `verdict.status = "PASS"` for a single run, then submit a run whose per-item results should have failed a gate. Run the `slo/verdict_fidelity_check.py` (which you also ship) that scans the warehouse for `(status=succeeded, aggregate_metric_below_gate_threshold=true, override=absent)` rows and confirms the mismatch surfaces. This is the correctness-SLO instrumentation catching a bug the vanilla monitoring would miss.

Both demonstrations produce short, reproducible commands with expected outputs recorded in `FAILURE_MODES.md`.

## Starter guidance

- **Author `SUBSTRATE.md` first.** The exercise is about the seams, not the pieces. Being explicit about which components are real and which are stubbed keeps the SLO numbers honest — a "reproducibility SLO of 99.9%" on a substrate whose plane is a stub is a fiction.
- **The verdict-is-opaque property is the single load-bearing invariant in Part B.** Every temptation to `if verdict.metrics["x"] < threshold` in the pipeline is the temptation that unbinds the gate registry from the pipeline. If you find yourself writing that logic in the pipeline, move it to the registry-hosted gate DSL.
- **Derive SLO targets before you tune the platform.** The whole point of the derivation script is that the target is a fact about the current platform, not an aspiration. Tune once you know where the P95 lives, not before.
- **Overrides need a real actor list.** A config file with two names is fine for the exercise, but the file is a real artifact — the `release-override` role is not "anyone with a keyboard." An open override role is worse than no override role.
- **The `bad-ci.yml` demonstration is educational, not defensive.** Do not try to make the bad CI job fail hard — the point of the side-by-side is to make the difference legible. A reader should be able to see the two files and know why the release-through-platform version is the one they want.
- **Fixture datasets for Part A must be registered.** The temptation is to `cat > /tmp/fixture.jsonl` in the CI job. Resist it: the fixture must go through the registry so that the smoke run's warehouse row has a resolved dataset hash. Otherwise the smoke run itself violates the lineage discipline the whole platform exists to enforce.
- **The `slo/report.py` is the artifact that convinces stakeholders the platform is a service.** Format it for a VP who has 60 seconds — headline adherence numbers first, per-SLO detail after, remediation notes at the end. A dashboard with 50 metrics that nobody reads is worse than a Markdown report with 5 that get read.

## Acceptance criteria

- The PR-level CI job runs schema validation and a fixture sanity-run under 5 minutes, fails closed on schema errors, and posts the diff-aware re-baseline comment when a judge with active-gate consumers is touched.
- The release-candidate pipeline submits through the platform, records the resolved suite hash in `release-audit.jsonl`, reads `verdict.status` only, and produces an audit line for every release-candidate run (pass, fail, and override-applied cases each demonstrated).
- The override path is an authenticated API call that writes to the warehouse, is consulted by the pipeline before failing, expires by default, and is queryable by `overrides_in_period`.
- `slo/definitions.yaml` declares all four SLOs with targets that were derived from the warehouse's own history (Part D's script produced the proposed targets); a reviewer can trace any target back to the query that produced it.
- `slo/report.py` produces a Markdown adherence report from the warehouse; the report's numbers match a hand-check on a small fixture warehouse.
- `slo/verdict_fidelity_check.py` catches the injected "green-light" bug from `FAILURE_MODES.md` demonstration 2.
- `FAILURE_MODES.md` demonstrates the two Chapter 6 anti-patterns with runnable commands and expected outputs.
- `SUBSTRATE.md` names which pieces are real vs. stubbed so the SLO numbers are honest.
- `CI.md` documents the fail-closed / fail-open policy for the PR-level job and the override process for the release pipeline.

## Stretch goals

- **Post-deploy scheduled runs.** Ship a cron-triggered job that submits the nightly regression suite from mod-110 Chapter 2 through the platform against the current production model. Wire the alert routing so a sequential-monitor alert from the observability platform (Arize / Langfuse / Weave) triggers an on-demand replay through the same API surface. This is the third CI altitude Chapter 6 names.
- **Error-budget-driven rollout freeze.** Ship an integration where the platform's `release-blocking` availability SLO burning through its monthly error budget triggers a rollout freeze on the release-train — no new candidates promoted until the budget replenishes. Requires a small `error_budget.py` computing the burn rate from the SLO adherence numbers and a hook into whatever the release-train tooling exposes. This is the SRE-workbook error-budget policy, applied to the eval platform.
- **Per-tenant SLO subscriptions.** Extend `slo/report.py` so each tenant can subscribe to its own latency and availability signals rather than the platform aggregate. A queue-priority misconfiguration or an adapter that specifically breaks one tenant's workload shape is invisible in aggregate; the per-tenant view is what surfaces it.
- **Public status page.** Ship a `/status` endpoint (a small HTML page or JSON blob is enough) that publishes the current SLO adherence in a form consumers can subscribe to. The Chapter 6 property is that adherence is published at the same cadence as the SLO — a monthly SLO with no monthly report is a fiction; make the report exist.
- **Replay through the platform.** Implement `platform.replay(run_id, override_model=?)` that reproduces a historical run against the same task/dataset/prompt/judge artifacts but a different model. This is the ad-hoc surface a post-deploy investigator uses to answer "did the metric shift come from the model or the input distribution?" — Chapter 6 mentions the replay pattern; a working implementation composes cleanly with exercises 02 and 04.
- **CI substrate portability.** Wire the same integration to *two* CI substrates (e.g., GitHub Actions and Argo Workflows). The interface — submit through the platform, read `verdict.status`, record audit — is portable by design; demonstrating it against two substrates is what proves the design didn't accidentally couple to one.
