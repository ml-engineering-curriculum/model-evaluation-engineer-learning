# exercise-05: Production Eval Observability Integration

**Estimated effort:** 2 hours

## Objective

Wire the outputs of the previous four exercises — offline regression scores, shadow deltas, A/B primary and guardrail metrics, and continuous-monitor alerts — into one of the three widely-adopted LLM observability platforms (Arize Phoenix, Langfuse, or W&B Weave) as a coherent traces-plus-evaluations stack. Enforce the Chapter 6 instrumentation contract on every LLM span (session id, user id, model version, prompt version, experiment arm, evaluator version), and design and validate the three drift-alert categories (input, output, metric) plus the judge-canary alert. The deliverable is a running local instance of the platform, a reproducible instrumentation harness that emits spans conformant to the contract, and a dashboard configuration (or a set of exported dashboard JSON) that a launch owner could actually read.

## Prerequisites

- mod-110 Chapter 6 (production observability and drift alerts), Chapters 2–5 for the metrics being fed in.
- mod-105 (LLM-as-judge configurations) — every evaluator attached to a span has a versioned judge behind it.
- Chapter 1 of the OpenInference spec / OpenTelemetry LLM semantic conventions for reference.
- Docker (or Podman) for running the platform locally, or a hosted account on one of the three platforms.
- Python 3.11+ with the platform SDK — `arize-phoenix` + `openinference-instrumentation-openai` for Phoenix; `langfuse` for Langfuse; `weave` for W&B Weave.

## Platform choice

**Pick one platform and commit for the duration of the exercise.** The exercise's goal is a coherent single-platform integration, not a comparison.

- **Arize Phoenix (`arize-phoenix`)** — recommended if you value OpenInference/OTel standardization and want the deepest instrumentation-library ecosystem. `pip install arize-phoenix`, run `phoenix serve` locally.
- **Langfuse (`langfuse`)** — recommended if you want a trace-first UI with a mature annotation queue and typed scoring model. `docker-compose up` from the Langfuse repo, or use the hosted free tier.
- **W&B Weave (`weave`)** — recommended if your team is already on W&B for training runs. `wandb login`, then `weave.init("project-name")`.

The rest of the exercise is written to be platform-agnostic; the deliverable adapts the shape to whichever platform you picked.

## Datasets

- A small **request corpus** (100–500 items) to replay through the instrumented model — reuse the shadow simulator's corpus from exercise-02 or a subset of `lmsys-chat-1m`. Each item has a stable `request_id`, a synthetic `session_id`, and a synthetic `user_id`.
- A pinned **judge-canary set** of 50–100 `(prompt, response, gold_label)` triples — reuse from exercise-04.
- A **drift-injection controller** — the corpus is fed through the instrumented harness three times: baseline, input-drift (shift the query-length distribution deliberately), output-drift (swap the model version silently mid-run). Used to demonstrate the three drift alerters.

## Requirements

### Part A — the instrumentation contract

Ship `instrumentation/contract.py` that defines the fields every LLM span must carry (Chapter 6):

- `session.id`, `user.id` (or hashed pseudonym), `model.name`, `model.version`, `experiment.id`, `experiment.arm`, `traffic.slice`, `prompt.template.id`, `prompt.template.version`.
- On evaluations: `evaluator.name`, `evaluator.version`, `evaluator.backend.model.version`.

Provide a `SpanContract` dataclass and a `validate_span(span_dict) -> list[str]` function that returns the list of missing contract fields. Ship a unit test that asserts an OpenInference-conformant span with all fields passes validation and that omitting any single field is caught.

### Part B — the instrumented harness

Ship `harness/replay.py` that:

- Reads the request corpus.
- Assigns synthetic `experiment.arm` and `traffic.slice` per request.
- Calls a target LLM (OpenAI, Anthropic, or a local `vLLM` endpoint) inside a traced span that emits all contract fields.
- Emits the span to the chosen platform (Phoenix / Langfuse / Weave) using the platform's SDK.
- Attaches an evaluation to each span using a judge from mod-105 (helpfulness 0–5), with the evaluator identity fields populated.

Include a smoke test that runs 10 requests, verifies every span landed in the platform via the platform's read API, and verifies every span passes `validate_span`.

### Part C — the three drift alerts

Implement three alerters that read from the platform (or an ETL side-file exported from it):

- **Input drift.** Compute PSI on the query-length histogram between the current window and a reference window. Alert (informational, no page) if PSI > 0.1; escalate to P3 investigate if PSI > 0.25. Chapter 6's thresholds.
- **Output drift.** Compute PSI on the output-length histogram and on the refusal rate between the current window and a reference window. Alert P3 if either shifts by an eyeballed threshold; escalate if it coincides with a metric-drift alert.
- **Metric drift.** Attach the exercise-04 confidence-sequence monitor to the helpfulness scores emitted as span evaluations. Alert P1 if the CS upper bound drops below the pre-committed floor; escalate per runbook.

Ship a `drift_alerters.py` module with these three alerters as callable classes; each has a `.check(window) -> AlertRecord | None`.

### Part D — the judge-canary integration

Reuse the exercise-04 judge-canary monitor, but instrument it inside the chosen platform:

- Register the canary dataset as a first-class platform artifact (Phoenix `Dataset`, Langfuse `Dataset`, or Weave `Dataset`).
- Run the current-production judge on the dataset on demand (Phoenix `phoenix.experiments`, Langfuse's `dataset_run`, Weave's `evaluate`), emitting evaluations tied to each dataset item.
- The judge-canary CS reads from those evaluations; when it alerts, the alert record links back to the platform's dataset-run view for root-causing.
- Include a mock-drift demonstration: swap the judge backend model version, re-run the dataset, and show the canary CS alerts within `k ≤ 3` runs.

### Part E — the dashboards a launch owner reads

Author (as platform-native dashboard configuration, exported JSON, or documented URLs) the five Chapter 6 dashboards:

1. **Always-on quality-and-safety dashboard.** One row per monitor with current value, CI, threshold, last-alert timestamp.
2. **Candidate-vs-incumbent view.** A/B primary metric and guardrails split by pre-declared segments, updated live from the platform.
3. **Judge-canary status page.** One row per production judge with agreement rate, CI, last vendor-model-version pin.
4. **Trace explorer.** A saved query in the platform filtered to a sample of production traces with evaluations inline. Not a summary — a queryable view.
5. **Dataset-and-eval library.** A listing of pinned datasets and evaluators with revision history.

Each dashboard is either a saved view in the platform or a shell script that opens the correct URL (Phoenix `phoenix.session.url`, Langfuse dashboard URL, Weave board URL). Ship a `dashboards/README.md` that maps each dashboard to the platform artifact and explains its intended audience.

### Part F — the alert-fatigue audit tool

Ship `audit/alert_review.py`:

- Reads the last N days of `AlertRecord`s from the platform (or from the alerter's log).
- Reports per monitor: fire count, true-positive rate (from a ground-truth injections file — for the exercise, the drift-injection labels from Part C), median time-to-diagnosis, actions taken.
- Recommends α adjustments and threshold retunings for monitors with < 20% true-positive rate (per Chapter 6's monthly-review discipline).

## Starter guidance

- **Commit to one platform before writing any code.** The three platforms diverge in API shape enough that adapting between them mid-exercise doubles the work. Pick one, commit for the duration.
- **The contract is the load-bearing artifact.** A dashboard that renders bad data is a bad dashboard; the contract is what keeps the data joinable. If your span validator does not fire on a span missing `experiment.arm`, the whole downstream A/B story is compromised.
- **Judge-canary before quality dashboards.** The single most valuable monitor in the fleet. If your production judge is drifting silently, every quality metric on every dashboard is lying, and you will attribute the lie to the target model.
- **Retention long enough to root-cause.** Phoenix and Langfuse default retention may be shorter than the longest realistic time-to-diagnose (a metric that starts drifting on Tuesday and pages on Friday). For the exercise, set retention to at least 30 days; document the setting in `dashboards/README.md`.
- **Every alert has a runbook.** Reuse the runbooks from exercise-04. A dashboard alert without a runbook is Chapter 6's alert-fatigue trap.
- **Do not print raw prompts or user IDs in exports.** Hash user IDs; keep raw prompts out of the alert records (they can live in the trace store under access controls). This is the mod-109 data-handling discipline applied to observability exports.

## Acceptance criteria

- One of Phoenix, Langfuse, or Weave is running locally (or hosted-account access is documented) and the harness successfully emits 100+ traces with the full contract.
- Every span emitted by the harness passes `validate_span` — no missing contract fields.
- The three drift alerters run over a controlled drift-injection replay and correctly identify the injected drift within one window (± the window size).
- The judge-canary integration fires within `k ≤ 3` runs when the judge backend model version is deliberately swapped.
- All five dashboards from Chapter 6 are documented in `dashboards/README.md` and each opens a working view (or, for the platform without an exact analog, the closest reasonable substitute is documented).
- The alert-fatigue audit tool produces a report over the injected drift log; monitors with < 20% TPR are flagged with recommended α or threshold adjustments.
- The instrumentation harness is reproducible — a second run of the same request corpus emits the same span identifiers (or an equivalent stable id) and the same evaluations.
- No raw user PII appears in exported alert records; user IDs are hashed and raw prompts stay behind the platform's access controls.

## Stretch goals

- **Multi-platform sanity comparison.** Emit the same 100-request replay to a *second* platform (e.g., you built the primary on Phoenix, mirror to Langfuse). Compare the trace / evaluation counts and reconcile any drift. This is a rare case where two-platform emission is worth the cost — as a one-time correctness check on your instrumentation, not as a permanent architecture.
- **OpenInference conformance test suite.** Take a public OpenInference conformance test (from the `openinference` repo) and run it against your harness's span output. Ship the passing report.
- **Prompt-versioning integration.** Wire your prompt library into Langfuse's `Prompt` primitive (or the equivalent on Phoenix / Weave). Show that a prompt change bumps the version, and the alerter treats the prompt change as an output-drift-suspicious event.
- **Sampling for scale.** Add a sampling layer to the harness (per-user hash-based sampling to reach a fixed sample rate). Document the sampling rate and prove it holds by counting spans over a fixed corpus size.
- **RBAC and retention policy.** Configure the platform's role-based access and per-project retention policy per your team's data-classification tier; document the settings in `dashboards/README.md`. This is where an "observability integration" becomes an "audited eval platform."
- **Judge-drift auto-response.** When the judge canary alerts, automatically post the affected downstream monitor list to a Slack / channel with a request to review re-baselining. The runbook drives the human decision; the auto-response ensures nobody misses the notification.
