# exercise-02: Shadow-Launch Comparison on Non-IID Traffic

**Estimated effort:** 2 hours

## Objective

Build a shadow-comparison analysis pipeline that consumes a `(request, incumbent_response, candidate_response)` triple stream from a shadow deployment (or a well-constructed simulation of one), computes paired quality and safety-classifier deltas with cluster-robust confidence intervals, stratifies the deltas by pre-declared axes, and issues a green / yellow / red verdict against a pre-registered decision rule. The exercise is not about running a real shadow deployment (that requires production infrastructure); it is about building the analysis and the reporting shape so that when a shadow deployment is available, the pipeline produces a defensible verdict.

## Prerequisites

- mod-110 Chapter 3 (shadow and dark-launch comparisons on non-IID traffic), Chapter 1 for altitude framing.
- mod-101 Chapter 2 (bootstrap CIs), Chapter 3 (paired comparisons), especially the cluster-bootstrap discipline.
- mod-105 Chapter on judge configurations — the quality delta in the shadow report is judge-graded.
- Python 3.11+ with `numpy`, `pandas`, and access to at least one LLM-as-judge (or a smaller trained classifier standing in).

## Datasets

You need a triple stream. Two sourcing options:

- **Simulated stream (recommended for this exercise).** Take an existing production-shaped dataset (e.g., `lmsys/lmsys-chat-1m` filtered to a manageable size, or `mteb`-style conversational data) and split it into `session_id`-tagged clusters. For each cluster, generate `incumbent_response` with model A (e.g., `Llama-3.1-8B-Instruct`) and `candidate_response` with model B (e.g., `Llama-3.1-70B-Instruct` or a fine-tuned variant). This is a *simulated* shadow — the responses were not from a live deployment — but the paired analytical shape is identical.
- **Real shadow data (if you have access).** A production shadow logging system that emits triples with `(request_id, session_id, user_id, timestamp, product_surface, incumbent_response, candidate_response, incumbent_serving_latency, candidate_serving_latency)`. Anonymize / hash IDs and strip PII before analysis.

Pin the dataset revision (or the simulation seed) in the report header. Aim for `n_sessions ≥ 500` and `n_requests ≥ 2000` for meaningful CIs.

## Requirements

### Part A — the input contract

Ship `shadow_analysis/schema.py` that defines the shadow-triple record shape:

```python
@dataclass
class ShadowTriple:
    request_id: str
    session_id: str
    user_id: str
    timestamp: datetime
    product_surface: str
    device: str          # e.g., "mobile" | "desktop"
    language: str        # e.g., "en" | "es" | ...
    tenure_days: int
    incumbent_response: str
    candidate_response: str
    incumbent_ttft_ms: float
    candidate_ttft_ms: float
    incumbent_completion_ms: float
    candidate_completion_ms: float
    incumbent_output_tokens: int
    candidate_output_tokens: int
```

The rest of the pipeline assumes this shape. A loader for both the simulated stream and the real-shadow stream normalizes to this schema.

### Part B — the evaluators

For each triple, compute:

- **Judge-graded quality metrics** on both responses using a mod-105 judge: helpfulness (0–5), correctness (0/1). Use the same judge configuration on both responses; do not swap judges between the two sides.
- **Safety-classifier flags** on both responses: refusal-detected (from a Llama-Guard-scale classifier), toxicity flag (from a `unitary/toxic-bert` or `Perspective` output when available), format-compliance flag (a regex or JSON-parse check when the request type expects structure).
- **Serving metrics** (already in the triple) passed through as-is; the pipeline reports the distribution, not the mean.
- **Divergence indicators.** For each pair, compute `edit_distance(inc, cand)` (or a normalized version), an embedding cosine similarity, and a categorical "did-they-agree-on-refusal" flag.

Every evaluator records its own version SHA and — for LLM judges — the backend model version.

### Part C — the paired cluster-bootstrap

Implement `paired_cluster_bootstrap(triples, cluster_key="session_id", n_boot=2000, seed=17)`:

- Aligns all `(inc, cand)` metric values into a per-triple list.
- Bootstraps over *clusters* — resample sessions with replacement, take all triples in each resampled session.
- Returns a delta CI for each metric.

Include a unit test that verifies the CIs widen appropriately when session-level correlation is introduced (feed the tester a dataset where all triples in a session share their metric value and verify the CI reflects the reduced effective sample size).

### Part D — stratification

Compute the delta separately for at least these strata:

- Weekday morning / weekday evening / weekend (derived from `timestamp`).
- Mobile / desktop / other (from `device`).
- English / non-English (from `language`).
- New (`tenure_days < 30`) / tenured.
- Product surface (as a categorical breakdown).

Every stratum reports `n_sessions`, `Δ`, `95% CI`, using the cluster bootstrap.

### Part E — heavy-tailed serving-metric reporting

For TTFT, completion latency, and output token count, do *not* report the mean alone. Report the full distribution (P50, P90, P95, P99) for both incumbent and candidate, plus the delta at each percentile. Reason: Chapter 3's "heavy tails" discussion — the mean and the P95 can point in different directions on serving metrics, and the launch review needs both.

### Part F — the pre-registered decision rule

Author a `SHADOW_PLAN.md` file *before* running the analysis on your candidate. It commits to:

- The metrics that will be reported (primary quality metric, secondary quality metrics, safety-classifier metrics, serving metrics).
- The strata that will be reported.
- The green / yellow / red thresholds — for example:
  - Green: aggregate primary CI lower bound ≥ 0 AND no stratum has a CI upper bound ≤ -0.02 on primary AND no safety-classifier metric has a CI lower bound ≥ +0.002.
  - Yellow: aggregate primary CI includes 0 OR one stratum shows a delta CI upper bound ≤ -0.02 OR serving P95 shift ≥ +100ms.
  - Red: aggregate primary CI upper bound ≤ 0 OR any safety metric CI lower bound ≥ +0.002.
- The window (start / end timestamps) and the expected traffic composition.

Store `SHADOW_PLAN.md` in the same repo as the analysis code; the analysis reads it and reports the verdict against the plan's rules, not against post-hoc rules.

### Part G — the report

`python -m shadow_analysis.report triples.jsonl --plan SHADOW_PLAN.md --out report.md` produces the report shape from Chapter 3 (window, cluster unit, quality metrics table, safety-classifier metrics table, serving metrics table, stratified deltas table, divergence indicators table, verdict per plan). The verdict block explicitly names each rule from the plan and whether it passed.

## Starter guidance

- **Simulate the stream if you do not have a real shadow.** The exercise is about analysis, not about production infrastructure. A well-constructed simulation from `lmsys-chat-1m` or a similar public conversational corpus, split by `conversation_id` into "sessions," produces a triple stream that exercises every part of the pipeline.
- **Cluster-bootstrap unit test.** The single most common bug in this pipeline is treating requests as independent when they belong to sessions. A synthetic dataset where every session's triples share a metric value is the sanity check — the CI on that dataset should be the CI on the number of sessions, not the number of requests.
- **Judges are stateful.** Two runs of the same judge on the same input can differ (especially at nonzero temperature). If you find judge variance affecting your deltas, either reduce temperature to 0.0 or `k=3` sample with majority vote.
- **The pre-registered plan is a real deliverable.** The single most common way this analysis lies is post-hoc mining ("we found a favorable subgroup"). Writing the plan before looking at the numbers is what makes the verdict defensible.
- **Percentiles for heavy tails, distributions in the report.** If your report shows "mean TTFT" and calls it a day, you are hiding the P95 shift that the SLA cares about.
- **Do not print raw prompts or responses in the report.** Individual triples, if quoted at all, are hashed. This is the mod-109 Chapter 7 data-handling discipline applied here.

## Acceptance criteria

- Every triple record loaded through the schema; missing required fields are rejected at load time with a clear error.
- The paired cluster-bootstrap function passes its unit test — CI widens correctly under session-level correlation.
- The report contains at least three strata beyond the aggregate, each with `n_sessions`, `Δ`, and `95% CI`.
- Serving-metric reporting includes P50, P90, P95, and P99 for both models plus the deltas at each percentile.
- `SHADOW_PLAN.md` exists and was written before the analysis was run (a git log check is the honest way to verify).
- The report ends with a verdict block that names each rule in `SHADOW_PLAN.md` and reports its pass/fail with the specific numbers that produced the verdict.
- No raw prompt or response text appears in the report; per-request log files live in a restricted-permissions directory.
- The pipeline is reproducible under a seed — the CIs and the verdict are byte-identical across two runs on the same input.

## Stretch goals

- **Adaptive re-weighting.** If your simulated stream's composition differs from your target production distribution (e.g., over-represents English), implement inverse-propensity weighting to re-weight the aggregate delta and compare weighted vs. unweighted results.
- **Divergence-mining tool.** For triples where the two models disagree significantly (edit-distance > threshold), cluster them by prompt topic (embedding K-means or an LLM-labeled clustering) and report which topics show the most divergence. This is where a shadow catches unexpected model behavior early.
- **SUTVA sanity check.** If your triples include a `turn_within_session` index, compare the delta on turn 1 versus later turns. A large gap suggests within-session interference is real for your product; report the finding.
- **Wire to an observability platform.** Emit the per-triple analysis outputs as evaluations attached to spans in Phoenix, Langfuse, or W&B Weave (Chapter 6). The report becomes a queryable artifact in the platform rather than a one-off markdown file.
