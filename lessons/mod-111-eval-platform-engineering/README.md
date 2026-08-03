# mod-111-eval-platform-engineering: Eval Platform Engineering: Eval-as-a-Service

**Estimated effort:** 16 hours

## Learning objectives

- Design an eval-as-a-service architecture with a versioned task / dataset / judge / prompt registry
- Embed multiple runners (lm-eval-harness, Inspect, OpenAI evals) behind a single orchestration plane
- Apply parallelism and cost controls (token budget, runner budget, judge-tier routing) at platform scope
- Stand up an eval data warehouse with traceable result lineage (model hash, dataset hash, judge hash, prompt hash, decoding params, seed)
- Integrate eval into CI and release pipelines and define SLA / SLO for the eval system itself

## Chapters

1. [Eval-as-a-Service and the Platform Mandate](01-eval-as-a-service-and-the-platform-mandate.md) — frames why individual evaluations grow into a platform, names the five layers the module builds (registry, orchestration, cost controls, warehouse, CI integration), and draws the boundaries with adjacent platforms (serving, observability, experiment tracking, CI/CD).
2. [A Versioned Registry for Tasks, Datasets, Judges, and Prompts](02-versioned-registry-for-tasks-datasets-judges-and-prompts.md) — the load-bearing foundation: four artifact types, content-addressed immutable revisions, semver conventions, deprecation lifecycle, and the migration strategy that actually ships.
3. [The Multi-Runner Orchestration Plane](03-multi-runner-orchestration-plane.md) — the user-facing submission surface, the runner-adapter contract, dispatch policy across lm-evaluation-harness / Inspect / OpenAI evals / internal harnesses, and the platform-owned model-client layer.
4. [Parallelism and Cost Controls at Platform Scope](04-parallelism-and-cost-controls-at-platform-scope.md) — the four cost axes, per-run and per-tenant token budgets, judge-tier routing, priority queues and reserved concurrency, rate-limit shaping, prompt caching, and cost attribution.
5. [The Eval Data Warehouse and Result Lineage](05-eval-data-warehouse-and-result-lineage.md) — the schema, the hash-based lineage graph, the four foundational queries (exact reproduction, metric-over-time, per-item regression diff, cost attribution), retention split between metadata and payload, and PII handling.
6. [Eval in CI, Release Pipelines, and the Platform's Own SLOs](06-eval-in-ci-release-pipelines-with-slo.md) — the three CI plug-in altitudes, the verdict-as-opaque-value pattern, override as a first-class API, and the four platform SLO categories (availability, latency, correctness, cost predictability) derived from warehouse history.

## Structure

- `01-…md` … `06-…md`: lecture chapters (above).
- `exercises/`: per-exercise prompts. Solutions live in the paired `-solutions` repo.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
