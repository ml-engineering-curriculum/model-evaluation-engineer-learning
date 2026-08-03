# exercise-04: Eval Data Warehouse and Lineage

**Estimated effort:** 3 hours

## Objective

Stand up the eval data warehouse from Chapter 5: the primary tables (`eval_run`, `run_item_result`, `run_aggregate_metric`, `run_event_log`, `run_cost`), the hash-based lineage graph that resolves through the Chapter 2 registry, idempotent ingestion, the four foundational queries (exact reproduction, metric-over-time, per-item regression diff, cost attribution), and a documented retention and PII policy. The deliverable is a working warehouse populated by the orchestration plane's events (from exercise-02) with all four queries returning correct results, and a written **retention & privacy policy** that the schema and ingest layer enforce.

## Prerequisites

- mod-111 Chapter 5 (warehouse and lineage). Chapter 2 (registry) and Chapter 3 (orchestration) for the sources of the events the warehouse ingests.
- mod-101 Chapter 4 (bootstrap CI) for the aggregation semantics `run_aggregate_metric.ci_low_95` / `ci_high_95` are computed from.
- Python 3.11+ with a warehouse substrate — Postgres for the metadata (via SQLAlchemy or SQLModel), or DuckDB / SQLite for a lighter-weight exercise. A local S3-compatible object store (`minio`, `moto`) or a directory for the payload store.

## Requirements

### Part A — the schema

Implement the schema from Chapter 5 verbatim (with modest naming adjustments for the substrate you choose). At minimum:

- `eval_run` with all fields from Chapter 5: `run_id`, `submitted_at`, `completed_at`, `requesting_team`, `workload_class`, `runner`, `runner_version`, `task_revision_id`, `model_revision_id`, `dataset_revision_id`, `prompt_revision_id`, `decoding_hash`, `seed`, `status`, budgets, `attribution`.
- `run_item_result` with `run_id`, `item_id`, `input`, `model_output`, `raw_score`, `normalized_score`, `judge_revision_id`, `judge_tier_actual`, `usage`, `latency_ms`, `scored_at`, `retries` and the composite primary key.
- `run_aggregate_metric` with `run_id`, `metric_name`, `metric_version`, `n`, `value`, `ci_low_95`, `ci_high_95`, `ci_method`, `aggregated_at`, `aggregation_hash`.
- `run_event_log` with `run_id`, `event_time`, `event_type`, `payload`.
- `run_cost` with `run_id`, `backend`, `input_tokens`, `cached_input_tokens`, `output_tokens`, `cost_usd_estimate`, `price_config_hash`.

Foreign keys `task_revision_id`, `dataset_revision_id`, `prompt_revision_id`, `judge_revision_id` all point into the registry from exercise-01 (or a stub table). `run_item_result.input` and `run_item_result.model_output` are stored either inline as JSONB (small payloads) or as object-store URIs (large payloads); document which threshold triggers the split.

### Part B — idempotent ingestion

Ship `warehouse/ingest.py` that consumes an event stream from the orchestration plane and writes to the warehouse:

- Every `item.completed` event is upserted into `run_item_result` keyed by `(run_id, item_id)`. A duplicate event is a no-op with a `duplicate_ingested` log entry.
- Every `budget.consumed`, `retry`, `error`, `dispatch_decision`, `queue.preempted`, `judge_tier.escalated`, `judge_tier.downgraded` event is appended to `run_event_log`.
- On `run.completed`, the ingester triggers the aggregation job (Part C).
- On `run.failed` or `run.budget_exceeded`, the ingester writes the terminal status but does not trigger aggregation.

The ingester tolerates out-of-order events (a `run.completed` that arrives before all its `item.completed` events retries the aggregation after a bounded wait) and duplicate events (idempotence).

### Part C — the aggregation job

Ship `warehouse/aggregate.py`:

- On `run.completed`, reads all `run_item_result` rows for the run and computes:
  - The mean (or task-appropriate aggregate) as `value`.
  - A bootstrap 95% CI (`ci_low_95`, `ci_high_95`) using `seed=17` and `n_boot=2000` — mod-101 Chapter 4's discipline. Writes `ci_method = "bootstrap_percentile"`.
  - An `aggregation_hash` computed over the canonicalized aggregation config (the metric definition, the bootstrap parameters, the aggregation code's git ref).
- Writes the row into `run_aggregate_metric`.
- Compares the re-computed aggregate to the runner-reported aggregate (from the adapter's `parse_results` output). If they diverge by more than a documented tolerance, appends a `aggregate_mismatch` event to `run_event_log`.

### Part D — the four foundational queries

Ship `warehouse/queries.py` with the four queries from Chapter 5, each returning typed results:

1. **`exact_reproduction(run_id)`** — returns a dict with `task_hash`, `dataset_hash`, `prompt_hash`, `judge_hash`, `model_hash`, `decoding_hash`, `seed`, `runner`, `runner_version` plus storage URIs. Used by `platform.rerun(run_id)`.
2. **`metric_over_time(task_revision_id, dataset_revision_id, metric_name, since, until)`** — returns a time series of `(completed_at, model_name, value, ci_low_95, ci_high_95)` rows. Refuses to return results if the WHERE clause does not pin `task_revision_id` and `dataset_revision_id` (raising an explicit error).
3. **`regression_diff(incumbent_run_id, candidate_run_id, top_k=100)`** — joins on `item_id`, returns the top-`k` items where the candidate scored lower than the incumbent. Requires the two runs' task hashes and dataset hashes to match; refuses to run otherwise.
4. **`cost_attribution(period_start, period_end)`** — returns `(requesting_team, total_runs, spend_usd, release_gate_runs, runs_that_blocked_a_release)` per team.

Ship at least one unit test per query with a fixture-populated warehouse.

### Part E — retention and PII policy

Write `RETENTION_AND_PII.md` and implement it:

- **Metadata retention.** The row-level tables (`eval_run`, `run_aggregate_metric`, `run_event_log`, `run_cost`) are retained on the platform's audit horizon — the exercise picks a number and documents it (say 3 years).
- **Payload retention.** The `input` and `model_output` columns of `run_item_result` are retained on a shorter horizon (say 90 days). Implement a scheduled job that nullifies these columns (or drops the object-store objects) past the payload horizon; write a test that confirms the aggregation and lineage queries still work on runs whose payloads have been dropped.
- **Gold audit sample.** A stratified 1–2% of runs have their payloads retained on the audit horizon; document the stratification and implement the sampling.
- **PII scrubbing at ingest.** Ship a scrubber module that runs on ingest and redacts email addresses, phone numbers, credit-card numbers (via Luhn-checkable patterns), and API-key-shaped strings. The scrubber logs every redaction with the `run_id` and `item_id` for audit.
- **Column-level encryption.** For the `input` and `model_output` columns, implement (or document, if your substrate doesn't support it) column-level encryption keyed to a key ID that is dropped on payload expiry. This is the "drop the key, drop the data" pattern.
- **Secrets scanner.** Optional but recommended: run `detect-secrets` or an equivalent scanner on payloads at ingest, quarantining items with high-confidence secret matches into a separate table that is access-restricted.

## Starter guidance

- **DuckDB is a reasonable substrate for the exercise.** It's in-process, supports JSON and JSONB-like columns, and is fast enough that the four queries run in milliseconds on fixture data.
- **Fixture-generate the run data.** You don't need to actually run evals to populate the warehouse for this exercise; a script that fabricates 30 runs with realistic distributions is enough to exercise all four queries.
- **Idempotence is easiest with a composite primary key.** `(run_id, item_id)` on `run_item_result` and unique constraints on `(run_id, metric_name)` make duplicate handling nearly free.
- **The aggregation hash matters.** Two aggregations of the same per-item data using different bootstrap methods produce different aggregate rows; the hash disambiguates them. Chapter 5 makes this point; a downstream reader that queries `run_aggregate_metric.value` should be able to filter by `aggregation_hash` to compare like-with-like.
- **Query 2's refusal-to-run-without-pinning is the most important safeguard.** A dashboard that silently mixes task revisions is the "trend line silently comparing different things" failure mode; make the query itself refuse.
- **Payload drop must not break aggregation.** The aggregation job reads `normalized_score`, not `input` / `model_output`. Verify this holds — a run whose payloads have been dropped should still be able to be re-aggregated (though not re-executed).

## Acceptance criteria

- The schema is implemented; the foreign keys resolve into the registry; the composite primary key on `run_item_result` prevents duplicate rows.
- The ingester is idempotent under duplicate events, tolerates out-of-order events, and logs `duplicate_ingested` when it drops a duplicate.
- The aggregation job computes bootstrap 95% CIs, writes the `aggregation_hash`, and emits `aggregate_mismatch` when runner-reported and re-computed aggregates diverge.
- All four foundational queries return correct results on a fixture-populated warehouse. Query 2 refuses to run without a pinned `task_revision_id` and `dataset_revision_id`; query 3 refuses to run when the two runs' task/dataset hashes don't match.
- The retention & PII policy is documented in `RETENTION_AND_PII.md`, the payload-drop job runs on schedule, and the aggregation / lineage queries continue to work on runs past the payload horizon.
- The PII scrubber redacts at ingest and logs every redaction for audit.
- A unit test demonstrates that a run's `exact_reproduction` result — task hash, dataset hash, judge hash, prompt hash, model hash, decoding hash, seed, runner, runner_version — is complete and each hash resolves through the registry to a storage URI.

## Stretch goals

- **Iceberg / lakehouse migration path.** Ship a second warehouse implementation on top of Apache Iceberg (via `pyiceberg`) or DuckDB's Iceberg support. Demonstrate that the same queries run against both substrates and produce the same results on the same data.
- **Aggregate recompute job.** Ship `warehouse/recompute.py`: given an updated `aggregation_hash`, recompute the aggregate rows for a historical range and write them under the new hash without touching the old rows. This is the "aggregates are recomputable; per-item is ground truth" property in operational form.
- **Column-level encryption with a real KMS.** Wire the column-level encryption to a real KMS (Vault, AWS KMS, GCP Cloud KMS) rather than an in-process key. Demonstrate that dropping a key ID makes the corresponding payloads unrecoverable.
- **Observability handoff.** Emit the same per-item events (from exercise-02's event schema) into an Arize Phoenix or Langfuse instance in parallel with the warehouse ingest. Demonstrate that a specific `run_id` is queryable in both — the warehouse for the analytical view, the observability platform for the interactive trace view — per Chapter 5's "compose, don't conflate" guidance.
- **Per-slice aggregation.** Extend the aggregation job to compute per-slice metrics from the dataset revision's `slice_definitions`. Store as `run_aggregate_metric_slice(run_id, metric_name, slice_id, ...)`. This is what the shadow / regression reports in mod-110 Chapter 3 depend on downstream.
