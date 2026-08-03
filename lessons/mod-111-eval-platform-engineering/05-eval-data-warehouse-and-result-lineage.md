# The Eval Data Warehouse and Result Lineage

An eval platform's value compounds when its historical results are queryable — not just "what score did this model get on this benchmark" but "how did this metric distribution move over the last twelve builds," "which items regressed vs. the previous release," "what did each team spend on judged runs last quarter," "which candidates from the fine-tuning experiment produced the top-decile helpfulness scores at the P95 latency budget." Each of those is a query, not an analysis project, when there is a warehouse with a real schema; each is a two-week analysis project when there isn't.

This chapter is about the shape of that warehouse: the primary tables, the hash-based lineage graph that makes every score reproducible from first principles, the join keys, the retention policy, and the specific queries the discipline is designed to make cheap. It composes directly with Chapter 2's registry (which is where the hashes come from) and Chapter 3's normalized per-item results (which is what gets ingested).

## What "lineage" means for an eval result

Every metric row in the warehouse has to answer a specific question: **if I hand you this number, can you exactly reproduce it?** The answer needs to be yes with no interpretation. Lineage is the set of fields that make it so.

A minimum lineage bundle for a metric row:

- **Model hash.** The exact model version served during the run. For registered internal models, the artifact-revision content hash from the model registry. For hosted-vendor models, the vendor's versioned model identifier (`anthropic:claude-sonnet-4-6@2026-02-01`) plus, where the vendor exposes it, the specific model-serving instance identifier the run landed on.
- **Dataset hash.** The Chapter 2 content hash of the dataset revision (not just the name-and-revision label; the hash of the actual bytes). Every row's `dataset_id` in the warehouse is a foreign key into `artifact_revision` from the registry.
- **Judge hash.** The Chapter 2 content hash of the judge revision that produced the score, which itself pins the backend model version, the rubric prompt hash, and the decoding config.
- **Prompt hash.** The Chapter 2 content hash of the task prompt template (distinct from the judge prompt, which is embedded in the judge hash).
- **Decoding params.** Temperature, top-p, top-k, max output tokens, stop sequences, presence/frequency penalties (where relevant). Stored as a canonicalized JSON blob whose own hash is the natural key.
- **Seed.** The random seed as it was actually consumed by the run — item sampling seed, judge-decoding seed (where applicable), any bootstrap seeds used in the per-run aggregation.

This bundle is Chapter 3's per-item schema plus the run-level aggregate. Every row in every result table carries the bundle explicitly or through a foreign key; there is no row without lineage.

## The warehouse's primary tables

A workable schema is small. Five to seven tables carry the vast majority of the queries.

```
eval_run                              -- one row per submitted eval run
  run_id                (uuid, pk)
  submitted_at          (timestamp)
  completed_at          (timestamp, nullable)
  requesting_team       (text)
  workload_class        (enum: release-blocking, safety-canary, ...)
  runner                (text)          -- runner adapter used
  runner_version        (text)          -- pinned runner version at dispatch
  task_revision_id      (fk -> artifact_revision.id)
  model_revision_id     (fk -> model_revision.id)
  dataset_revision_id   (fk -> artifact_revision.id)
  prompt_revision_id    (fk -> artifact_revision.id)
  decoding_hash         (text)          -- canonicalized decoding config hash
  seed                  (bigint)
  status                (enum: succeeded, failed, cancelled, budget_exceeded)
  budget_tokens         (bigint)
  budget_wallclock_min  (int)
  attribution           (jsonb)         -- release_candidate, ticket, ...

run_item_result                       -- one row per (run, item)
  run_id                (fk)
  item_id               (text)         -- stable id within dataset revision
  input                 (jsonb)
  model_output          (jsonb)
  raw_score             (jsonb)
  normalized_score      (jsonb)        -- platform-canonical shape
  judge_revision_id     (fk -> artifact_revision.id, nullable)
  judge_tier_actual     (enum: tier0, tier1, ...)
  usage                 (jsonb)        -- prompt_tokens, completion_tokens, cached_tokens
  latency_ms            (int)
  scored_at             (timestamp)
  retries               (int)          -- retry count for this item
  PRIMARY KEY (run_id, item_id)

run_aggregate_metric                  -- one row per (run, metric)
  run_id                (fk)
  metric_name           (text)
  metric_version        (text)         -- registered metric revision
  n                     (int)
  value                 (double)
  ci_low_95             (double)
  ci_high_95            (double)
  ci_method             (text)         -- bootstrap, wilson, ...
  aggregated_at         (timestamp)
  aggregation_hash      (text)         -- hash of the aggregation code + config
  PRIMARY KEY (run_id, metric_name)

run_event_log                         -- one row per lifecycle event
  run_id                (fk)
  event_time            (timestamp)
  event_type            (text)         -- queued, started, item_completed, error, ...
  payload               (jsonb)

run_cost                              -- one row per (run, backend)
  run_id                (fk)
  backend               (text)         -- anthropic, openai, internal-vllm, ...
  input_tokens          (bigint)
  cached_input_tokens   (bigint)
  output_tokens         (bigint)
  cost_usd_estimate     (double)
  price_config_hash     (text)         -- hash of the pricing config used to estimate
  PRIMARY KEY (run_id, backend)
```

Two structural properties matter:

- **Per-item results are the ground truth; aggregates are derived.** `run_aggregate_metric` is a projection over `run_item_result`. Recomputing aggregates from per-item scores is a query; a new aggregation strategy is a new metric revision, not a mutation.
- **Every foreign key resolves to a content-addressed artifact revision.** `task_revision_id`, `model_revision_id`, `dataset_revision_id`, `prompt_revision_id`, and `judge_revision_id` all point into the Chapter 2 registry. Given a `run_id`, walking the graph produces the exact byte-level provenance of every score.

## Content-addressed lineage: the property that reproduction depends on

Every downstream question about reproducibility eventually reduces to the same walk of the lineage graph:

```
result row
  -> run_id
     -> task_revision_id       -> registry -> content_hash + storage_uri
        -> dataset_revision_id -> registry -> content_hash + storage_uri
        -> prompt_revision_id  -> registry -> content_hash + storage_uri
     -> model_revision_id      -> model registry -> weights / vendor version
     -> judge_revision_id      -> registry -> content_hash + storage_uri
        -> backend model version + rubric prompt hash (nested)
     -> decoding_hash          -> parameter bundle
     -> seed
     -> runner + runner_version
```

Given a metric row, this walk retrieves every byte the run consumed. That property is what makes the platform's SLO on correctness (Chapter 6) achievable — "we can re-run and get the same number" is a mechanical guarantee once the lineage is complete, not a "we'll try our best" aspiration.

The failure mode when the property is missing is characteristic: a report from three months ago shows a "helpfulness score of 4.3" on `helpfulness-judge-v2` and no one can tell whether that judge was the June revision or the August revision. If the row does not name the judge revision *content hash*, the walk terminates in ambiguity and the number is unfalsifiable.

## Choosing the storage substrate

The warehouse is a logical entity — a set of tables and a query engine. Its physical substrate depends on the organization's data-platform posture. Three shapes recur.

- **Postgres for the metadata, object store for the payloads.** The run/item/aggregate tables live in Postgres (or a managed equivalent — Aurora, CloudSQL); the input, output, and raw score payloads live in an object store (S3, GCS) with references from Postgres. This is the pragmatic default for platforms up to O(10⁸) items; queries are cheap, joins are cheap, and payload storage scales independently of query cost.
- **Data lake / lakehouse (Iceberg, Delta, Hudi) for everything.** Parquet files partitioned by date and workload class, cataloged in a lakehouse table format, queried through Trino, Athena, DuckDB, or Spark. Appropriate at higher scale (O(10⁹) items and above) and when the eval warehouse needs to compose easily with the organization's broader data lake.
- **Managed observability platform as substrate.** Arize Phoenix, Langfuse, and W&B Weave (mod-110 Chapter 6) can host the raw payloads and per-item scores natively. Attractive for small teams already using one of these platforms; less attractive at large scale because the warehouse-shape query patterns (multi-quarter trend analysis, per-release comparison) are not the platforms' primary use case.

The substrate choice is downstream of the schema, not upstream. Get the schema right and the substrate can be swapped without redesigning the eval discipline.

## The four foundational queries

The warehouse pays for itself when a small number of queries become cheap. Four are worth calling out because they compose to answer most business questions.

### Query 1: exact reproduction

```
-- Given a run_id, retrieve everything needed to re-run it byte-for-byte.
SELECT
  r.run_id,
  ar_task.content_hash    AS task_hash,     ar_task.storage_uri    AS task_uri,
  ar_ds.content_hash      AS dataset_hash,  ar_ds.storage_uri      AS dataset_uri,
  ar_pr.content_hash      AS prompt_hash,   ar_pr.storage_uri      AS prompt_uri,
  mr.name AS model_name,  mr.content_hash   AS model_hash,
  r.decoding_hash, r.seed, r.runner, r.runner_version
FROM eval_run r
JOIN artifact_revision ar_task ON ar_task.id = r.task_revision_id
JOIN artifact_revision ar_ds   ON ar_ds.id   = r.dataset_revision_id
JOIN artifact_revision ar_pr   ON ar_pr.id   = r.prompt_revision_id
JOIN model_revision    mr      ON mr.id      = r.model_revision_id
WHERE r.run_id = :run_id;
```

This is the query that satisfies the platform's correctness SLO. A run for which this query returns fully-populated hashes and storage URIs can be re-executed and expected to produce byte-identical results (up to the intended non-determinism captured in the seed).

### Query 2: metric-over-time on the same eval

```
-- How has this task's metric moved across the last N candidate models?
SELECT
  r.completed_at,
  mr.name AS model_name,
  agg.value,
  agg.ci_low_95,
  agg.ci_high_95
FROM eval_run r
JOIN run_aggregate_metric agg ON agg.run_id = r.run_id AND agg.metric_name = :metric
JOIN model_revision mr        ON mr.id = r.model_revision_id
WHERE r.task_revision_id  = :task_revision_id   -- pinned to a single task revision
  AND r.dataset_revision_id = :dataset_revision_id
  AND r.judge_revision_id_hint = :judge_revision_id  -- if judged
  AND r.status = 'succeeded'
ORDER BY r.completed_at DESC
LIMIT 30;
```

This is the query the re-baseline discipline of mod-110 Chapter 2 consumes: the incumbent's rolling distribution on a fixed task revision, from which quality-gate thresholds are derived. The critical property is that the WHERE clause pins the task, dataset, and judge revision hashes — anything else and the trend line is comparing across a metric-definition change.

### Query 3: per-item regression diff between two runs

```
-- Where did the candidate regress vs. the incumbent, item-by-item?
SELECT
  incumbent.item_id,
  incumbent.normalized_score AS incumbent_score,
  candidate.normalized_score AS candidate_score,
  (candidate.normalized_score->>'value')::double - (incumbent.normalized_score->>'value')::double AS delta
FROM run_item_result incumbent
JOIN run_item_result candidate
  ON candidate.item_id = incumbent.item_id
WHERE incumbent.run_id  = :incumbent_run
  AND candidate.run_id  = :candidate_run
  AND (candidate.normalized_score->>'value')::double < (incumbent.normalized_score->>'value')::double
ORDER BY delta ASC
LIMIT 100;
```

This is the query that turns "there's a regression" into "here are the 100 items driving the regression." It is a natural join on `item_id`, which is stable across runs of the same dataset revision. Chapter 6's CI reports use this to surface the top regressions in the human-readable report.

### Query 4: cost attribution by team and outcome

```
-- What did each team spend on eval, and what fraction of runs blocked a release?
SELECT
  r.requesting_team,
  COUNT(*)                              AS total_runs,
  SUM(rc.cost_usd_estimate)             AS spend_usd,
  SUM(CASE WHEN r.workload_class = 'release-blocking' AND r.status = 'succeeded' THEN 1 ELSE 0 END) AS release_gate_runs,
  SUM(CASE WHEN blocked.blocked_at IS NOT NULL THEN 1 ELSE 0 END) AS runs_that_blocked_a_release
FROM eval_run r
JOIN run_cost rc         ON rc.run_id = r.run_id
LEFT JOIN release_gate_decision blocked ON blocked.run_id = r.run_id AND blocked.decision = 'block'
WHERE r.submitted_at >= :quarter_start AND r.submitted_at < :quarter_end
GROUP BY r.requesting_team
ORDER BY spend_usd DESC;
```

This is the finance-conversation query from Chapter 4. When spend is broken down by team and cross-joined with the outcome of the runs (did any of them actually block a release; did any of them produce a signal that changed a decision), the conversation about the platform's ROI is decidable rather than vibe-based.

## Ingestion and idempotence

The warehouse ingests from Chapter 3's orchestration plane: the adapter emits per-item events, the plane serializes them to the warehouse. Two ingestion properties keep the data trustworthy.

- **Idempotence on `(run_id, item_id)`.** An item event that is emitted twice — because the adapter retried, because the event queue redelivered — produces a single row. Duplicate ingestion is a normal operational event, not a data-corruption event.
- **Aggregate rows are recomputed, not written.** `run_aggregate_metric` is populated by an aggregation job that reads from `run_item_result` and writes the aggregate row atomically at run completion. Aggregates are also re-computable on demand — if the metric's aggregation code is updated (a new bootstrap method, a bug fix), a job can recompute historical aggregates and write a new metric revision without touching the item rows.

An important corollary: **the per-item score, not the aggregate, is what the platform commits to.** Aggregates can be redone; per-item scores cannot. The lineage discipline applies to the per-item level; the aggregate is a projection.

## Retention: what to keep and for how long

Two retention horizons apply to different categories of data:

- **Metadata (run, item score, aggregate, cost, event log) is retained on the platform's audit horizon.** Typically 2–7 years, driven by regulatory posture (EU AI Act obligations for GPAI systems, contract retention clauses, model-card retention commitments). Metadata is cheap to keep; deleting it destroys the lineage story on which the platform's correctness claim depends.
- **Payload data (input, model_output, raw judge output) has a shorter retention.** Typically 90 days to 2 years, driven by storage cost and — importantly — by data-privacy obligations. When production replay is used as an eval dataset, the payload data is user data and its retention is constrained by the same policies that constrain the production data.

Payload retention introduces a subtle reproducibility problem: if the input payload has been aged out, the run cannot be re-executed even with all the artifact hashes. Chapter 6's SLO on correctness has to accommodate the retention policy — "reproducible within the payload retention window" is what is achievable, not "reproducible forever."

Practical mitigation: keep a small "gold audit sample" of the payload data (a stratified 1–2% sample) for a longer horizon. This lets audits verify representative runs without keeping every user input in storage indefinitely.

## PII, secrets, and the payload column

`run_item_result.input` and `run_item_result.model_output` can contain user-generated PII, model-generated PII, and vendor-specific credentials that leaked into a prompt. Three practices contain the risk.

- **Scrub before storage, not on egress.** Whatever PII policy the platform commits to (email addresses hashed, phone numbers redacted, credit-card numbers rejected outright) is enforced at ingest time. A scrub layer that runs on query would be a policy-violation waiting for the wrong query to run.
- **Column-level encryption on payload columns.** The metadata columns (`run_id`, `dataset_revision_id`, hashes, aggregate metrics) are queryable in the clear; the payload columns are encrypted at rest with keys held by a smaller access group. This composes with the retention policy — expired payloads are dropped by dropping their keys.
- **A separate secrets scanner.** Judge outputs occasionally emit secrets when a system prompt contains a template that the model completes with a plausible-looking API key. A scanner that runs on ingest and quarantines items with high-confidence secret matches (via `detect-secrets`, TruffleHog patterns, or a bespoke rules set) keeps them out of the general table.

Payload columns are where privacy incidents happen. The rest of the warehouse's shape is unremarkable from a privacy perspective, but the payload column deserves specific paranoia.

## Composition with observability

The observability platform (mod-110 Chapter 6 — Phoenix, Langfuse, Weave) and the warehouse serve different consumers and should not be conflated:

- **Observability serves the interactive-debug loop.** An engineer is investigating a specific trace, comparing two candidate outputs, or looking at judge rationales in-line with a scored run. Fast, indexed, narrow.
- **The warehouse serves the analytical and audit loop.** A quarterly cost review, a re-baseline calculation, a lineage audit for a shipped model. Cheap on large-scale aggregations; less flashy per-item.

The two share sources. The orchestration plane emits per-item events *both* to the warehouse (for durable analytics) *and* to the observability platform (for interactive traces). Some organizations use the observability platform's own storage as the warehouse; that works up to a scale and then bends. The clean pattern is to keep them coupled at the event stream but separate at the storage layer.

## Failure modes the warehouse is designed to prevent

### Failure mode: "we can't reproduce that number from Q2"

The launch review three months ago quoted a helpfulness score of 4.3. Reproducing it now requires reading old CI logs, finding the model version, discovering that the judge prompt has been edited twice since then, and hand-reconstructing the exact configuration. It cannot be done in a reasonable time and the number is quietly retracted from the ongoing narrative.

Warehouse mitigation: Query 1 above returns everything needed to re-run byte-for-byte, and the retention policy has kept the payloads within the audit horizon. Reproduction is `plane.rerun(run_id)`; correctness is a query, not an investigation.

### Failure mode: the trend line silently compares different things

A dashboard shows helpfulness score climbing from 3.8 to 4.3 over the last quarter. The rise is real for the first four data points and artifactual for the next six — the judge prompt was edited between build 5 and 6 in a way that adds 0.5 on average. The dashboard has been silently comparing across metric-definition changes.

Warehouse mitigation: Query 2 pins the judge revision hash in the WHERE clause. A trend line that mixes revisions is a UI bug (the caller passed a wildcard); the platform's default dashboards refuse to render such mixes.

### Failure mode: aggregate mismatch between the runner and the platform

The runner reports MMLU accuracy = 0.82; the warehouse's re-aggregation from per-item scores gives 0.81. The discrepancy is small enough to be silently ignored but persistent enough to be a symptom of an adapter bug that mis-normalizes a class of items.

Warehouse mitigation: the aggregation job compares runner-reported and re-computed aggregates on every run and emits a `run_event_log` entry when they disagree by more than a documented tolerance. Persistent mismatches surface as a platform lint; they are not silent.

## Guidance for the platform engineer

- **Content hash on every lineage foreign key.** `dataset_revision_id`, `judge_revision_id`, `prompt_revision_id` all point into `artifact_revision.content_hash`. A lineage story built on names alone will fail the first time an artifact is edited in place.
- **Per-item score is ground truth; aggregate is derived.** Aggregates are recomputable; per-item scores are not. The retention policy should reflect that distinction.
- **Idempotent ingest on `(run_id, item_id)`.** Duplicate events are a normal operational fact; the schema is unbothered by them.
- **Separate metadata and payload retention.** Metadata lives for the audit horizon; payloads live for the privacy horizon. A gold-audit sample bridges the gap.
- **Scrub PII at ingest, not on egress.** The privacy incident happens the moment the data lands; egress-time scrubbing is a policy hope.
- **Composition with observability, not conflation.** The observability platform is for interactive debug; the warehouse is for analytics and audit. Feed both from the same event stream, store separately.
- **Aggregate mismatch is a lint.** Runner-reported and re-computed aggregates should agree; when they don't, the platform surfaces it rather than swallowing it.

## Summary

The eval data warehouse is the second load-bearing platform layer after the registry. Its schema is small — `eval_run`, `run_item_result`, `run_aggregate_metric`, `run_event_log`, `run_cost` — and every foreign key resolves through Chapter 2's registry to a content-addressed artifact revision. Every metric row carries the full lineage bundle (model hash, dataset hash, judge hash, prompt hash, decoding params, seed) so that reproduction is a query rather than an investigation. Per-item results are the ground truth; aggregates are derived and recomputable. Retention splits into a long metadata horizon and a shorter payload horizon; a gold audit sample bridges them. PII scrubbing happens at ingest; payload columns get column-level encryption; secrets scanning quarantines high-confidence leaks. The warehouse composes with — does not replace — the observability platform for interactive debug. Four query shapes pay for the warehouse's existence: exact reproduction, metric-over-time on fixed revisions, per-item regression diff between two runs, and per-team cost attribution cross-joined with outcomes. The next chapter is about how CI systems consume the platform, how release pipelines block on eval verdicts, and what SLOs the eval platform owes to its consumers.
