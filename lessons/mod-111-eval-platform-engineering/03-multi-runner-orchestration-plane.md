# The Multi-Runner Orchestration Plane

Every mature eval program ends up with more than one runner. lm-evaluation-harness (EleutherAI's harness — the reference implementation for the Language Model Evaluation Harness ecosystem) is the pragmatic default for capability benchmarks with fixed answer keys. UK AISI's Inspect is the reference for solver/scorer pipelines with tool use, agentic scaffolds, and multi-step scoring. OpenAI evals is the reference for LLM-graded and model-graded task shapes and is common inside organizations that adopted it early. HELM's `helm-benchmark` is common in organizations doing HELM-style multi-metric analyses. Then there are internal harnesses — the retrieval-eval script the RAG team maintains, the agent-trajectory scorer the agent team wrote, the custom safety-eval driver the safety team owns.

You do not consolidate on one runner. You build an orchestration plane over all of them. This chapter is about the shape of that plane: what abstraction it exposes to users and CI systems, what adapter contract it enforces on the runners, how it composes with the registry from Chapter 2, and what specifically it does not attempt.

## Why not just pick one runner

A recurring temptation is to standardize on a single runner and force every eval into that runner's task shape. It is worth naming why this fails.

- **Runners have real capability differences.** lm-evaluation-harness is excellent at multiple-choice, log-likelihood, and generation-with-exact-match tasks; it is not the natural place to run a five-step agentic trajectory with a sandboxed tool-use scorer. Inspect is designed for exactly that; it is more mechanism than you want for a fixed-answer log-likelihood benchmark. Cramming Inspect-shaped tasks into lm-eval-harness (or vice versa) produces adapters that reimplement the destination runner poorly.
- **Runners have different community defaults.** The lm-eval-harness leaderboard number for MMLU has a specific prompt template, specific few-shot policy, and specific normalization; the HELM MMLU number has different ones. Both are correct against their own methodology; forcing one runner to reproduce the other's number is a losing multi-month project.
- **Adopting a runner is a community investment, not a technology choice.** The team that wrote the RAG eval script in Inspect used Inspect because Inspect's solver/scorer primitives matched how they think about RAG. Moving them to lm-eval-harness because "we picked one runner" trades an evaluation problem for a coordination problem.

The correct primitive to standardize on is *the interface to the platform*, not the runner. Users request an eval by (task, model, config) — the orchestration plane picks the runner.

## What the orchestration plane exposes to users

The plane's user-facing surface is small on purpose. A minimal shape:

```
POST /v1/eval-runs
{
  "task": "safety.harmbench_standard@1a2b3c4",     -- registry ref (Chapter 2)
  "model": "internal/candidate-v0.14.2",           -- registry ref (or model-serving ref)
  "config": {
    "seed": 17,
    "sample_size": 300,
    "priority": "release-blocking",
    "budget_tokens": 2_500_000,
    "budget_wallclock_min": 45
  },
  "attributions": {
    "requesting_team": "safety-eval",
    "release_candidate": "release-2026-q3-w28"
  }
}
```

The plane accepts this request, resolves the task through the registry, picks a runner, allocates a budget from Chapter 4's controls, and returns an `eval-run-id`. Downstream operations (status polling, result retrieval, cancellation) target the run-id. The user does not know — and does not need to know — that this particular task translates to an lm-evaluation-harness invocation on a specific runner replica pool.

Three properties of this surface matter:

- **Registry references, not file paths.** The task, dataset, judge, and prompt references are resolved through the Chapter 2 registry. The user cannot smuggle in an unversioned artifact; the plane rejects requests that don't resolve.
- **Model references are versioned too.** Whether the model lives in an internal model registry (Chapter 2-style artifact) or in a serving platform's route (`vllm:/models/candidate-v0.14.2`), the reference must carry an unambiguous version. Unversioned vendor routes (`openai/gpt-4`) are the same failure the judge story in Chapter 2 rejected; the plane enforces the same rule for models under test.
- **Priorities and budgets are declared, not inferred.** Chapter 4 walks the priority classes and budget mechanics; the plane exposes them so that release-blocking runs get the queue slot they need without the requesting team having to know how the queue works.

## The runner adapter contract

For each runner the platform supports, an adapter implements a small interface. In the pattern this chapter recommends, the adapter is a stateless process (usually containerized) that the plane can invoke and monitor. The adapter must:

- **Translate a registry-resolved task into the runner's native task shape.** The lm-evaluation-harness adapter writes a YAML config that lm-eval-harness understands; the Inspect adapter writes a Python file (or in-memory task) that the Inspect CLI can consume; the OpenAI-evals adapter writes the corresponding eval YAML.
- **Enforce configuration invariants.** The adapter refuses to run if the runner's config diverges from the registered task (a random temperature, an unversioned model route, a swapped-out prompt). This is the trust boundary — anything the runner is asked to do must be traceable through the platform.
- **Stream a structured event log.** As the runner executes, the adapter emits events on a common schema: `run.started`, `item.completed`, `partial_score`, `budget.consumed`, `error`. The plane consumes these to update run status, meter budgets, and route alerts.
- **Emit results on a common schema.** When the runner finishes, the adapter emits per-item and aggregate results into the plane's result queue. The schema is fixed at the platform layer (Chapter 5 walks it in detail); every adapter conforms.
- **Report lineage on completion.** The adapter attaches the runner's own version, the runner's dependency snapshot (a pip freeze or an image digest), the model version actually served, the environment fingerprint (GPU generation, driver version), and the seed as it was actually consumed.

The adapter is not a translation of the *user experience* of each runner — the plane does not try to make Inspect look like lm-eval-harness. It is a translation of the *invocation contract*: whatever the runner is, the plane can start it, monitor it, meter it, and reap its output.

### A concrete adapter shape (pseudocode)

```python
class RunnerAdapter(Protocol):
    name: str                                  # "lm-eval-harness", "inspect", "openai-evals"
    supported_task_shapes: list[str]           # {"multiple_choice", "generation", "trajectory"}

    def can_run(self, resolved_task: Task) -> bool:
        """Registry-resolved task -> True/False for this runner."""

    def prepare(
        self,
        resolved_task: Task,
        resolved_model: Model,
        run_config: RunConfig,
    ) -> RunnerInvocation:
        """Materialize the runner's native config, staged artifacts, and command line."""

    def execute(
        self,
        invocation: RunnerInvocation,
        event_sink: EventSink,
        budget: BudgetHandle,
    ) -> RunnerResult:
        """Run the runner. Emit events to event_sink, respect budget.decrement(...)."""

    def parse_results(
        self,
        raw: RunnerResult,
    ) -> Iterable[PerItemScore] | AggregateScore:
        """Normalize into the platform's result schema."""

    def lineage_snapshot(self, invocation: RunnerInvocation) -> Lineage:
        """Runner version, dep hash, env fingerprint, model version actually served."""
```

This shape is deliberately not aiming for a "least common denominator" of eval semantics. It aims for a common execution and lineage surface; the semantics stay in the runner.

## The dispatch decision: choosing a runner for a task

The plane needs a policy for picking a runner when more than one can execute a task. Three inputs feed the decision:

- **Task-declared runner compatibility.** The registered task carries a list of runners that can execute it. For most tasks this is a single runner (chosen at registration); for some it is multiple (a HarmBench-shape refusal eval can run in Inspect or in a custom harness). Tasks that assert "any capable runner" are rare and usually a smell.
- **Runner health and current backlog.** The plane tracks each runner's current queue depth and rolling error rate. A backlog on the lm-eval-harness pool routes a request that could run in either lm-eval-harness or an internal harness to the internal harness. Under Chapter 4's priority classes, release-blocking runs get first pick.
- **Team preference or override.** For tasks with multiple compatible runners, the requesting team can pin — "always run this with Inspect" — either at registration or at request time. Silent runner switches for the same task would break comparability across runs; the plane makes any switch explicit.

The dispatcher's decision is logged with the run's lineage. When a report shows a metric that surprised someone, "which runner did this run on" is one of the first questions and it must be a query, not an investigation.

## Model plurality: the model side of orchestration

Symmetric to the runner story, the plane usually calls many *model* backends: hosted vendors (Anthropic, OpenAI, Google, Together), self-hosted inference stacks (vLLM, SGLang, TGI), and internal experimental checkpoints. The plane abstracts these behind a **model client** interface that adapters call rather than instantiating clients directly.

Key properties of the model client layer:

- **A common request/response schema.** Every model call goes through the same shape: `(messages | prompt, decoding_config, stop_sequences, seed?)` in, `(text, usage, finish_reason)` out. Vendor-specific bells and whistles (system-prompt caching hints, prompt-caching TTLs, structured output parsers) are exposed as optional fields but do not leak into the adapter code.
- **A single retry, backoff, and rate-limit policy.** Chapter 4 walks the platform-scope quota story; the model client is where that policy lands operationally. Individual adapters do not implement their own retries; if they did, cross-team quota accounting would be impossible.
- **Instrumentation on every call.** Every model invocation emits a span with the input token count, output token count, wall-clock latency, cost estimate, cache-hit indicators, and the exact model version served. Chapter 5's warehouse joins on these; without them, cost accounting and lineage both break.
- **Versioning enforcement.** The client rejects `provider:model` routes without a version qualifier. `anthropic:claude-sonnet-4-6@2026-02-01` is accepted; bare `anthropic:claude-sonnet-4-6` is not, unless the platform has an explicit alias that resolves to a versioned target.

Frameworks like LiteLLM, Portkey, and cloud-specific gateways (Vertex AI, Bedrock) can serve as the model-client layer's substrate — but the platform's own thin wrapper on top is what enforces the versioning, quota, and lineage properties above. Delegating to the underlying gateway wholesale means inheriting the gateway's version-pinning story, which is usually laxer than the platform needs.

## Composition with the registry

The registry (Chapter 2) is what the plane resolves against. Every eval-run request follows this pattern:

1. Parse the request, extract registry references.
2. Resolve each reference to a specific content-addressed revision. Unresolvable references (deprecated-to-archived, typos) fail here with a clear error.
3. Consult the registered task's runner compatibility. If more than one runner qualifies, apply the dispatch policy above.
4. Assemble the resolved artifacts into the selected runner's adapter, allocate a budget (Chapter 4), and queue the invocation.
5. On completion, write the resolved artifact hashes into the result row (Chapter 5). Every result knows what it is a measurement of.

This flow is worth calling out because a subtly-different flow — where the runner reaches into the registry itself — creates a lineage hole. If the adapter code is what resolves references, adapters that behave slightly differently can produce differing resolutions for the same request, and the "what did this run measure" question loses its clean answer. Resolution happens once, at the plane, before dispatch.

## Result normalization: what "the same score" means across runners

Different runners report scores in different shapes. lm-eval-harness returns a per-task aggregate metric with an internal SE estimate; Inspect returns per-item Score objects with optional metadata; OpenAI evals returns a JSONL of records that a downstream aggregator has to sum. The plane's job is to normalize these into a single per-item schema in the warehouse:

```
run_item_result
  run_id            (fk)
  item_id           (text)         -- stable identifier within the dataset revision
  input             (text/jsonb)
  model_output      (text/jsonb)
  raw_score         (jsonb)        -- runner-native score object
  normalized_score  (jsonb)        -- platform-canonical score for this metric type
  scored_by         (fk -> judge revision)
  scored_at         (timestamp)
  usage             (jsonb)        -- prompt/completion tokens, cost estimate
  lineage           (jsonb)        -- runner version, model version, seed
```

Aggregation over `run_item_result` is a query, not a runner-specific step. That property is what lets Chapter 5's warehouse compute the confidence intervals from mod-101 Chapter 2 uniformly across runners — the CI is not a function of the runner; it is a function of the per-item score column. If a runner's native aggregate disagrees with the platform's re-aggregation from per-item scores, that is a bug in the adapter, and it is one the platform can detect with a simple invariant check.

## Failure modes the orchestration plane is designed to prevent

### Failure mode: runner-CLI drift silently changes eval semantics

A capability team upgrades their local lm-eval-harness from `0.4.4` to `0.4.5` because a colleague suggested it. The `hellaswag` task in `0.4.5` has a slightly different prompt template; every score from the two versions is subtly incomparable. The team's leaderboard shows a "+1.2" bump that is entirely artifactual.

Plane mitigation: the runner version is pinned per-adapter and reported in every run's lineage. Upgrading the pin is a platform change, not a per-team change, and every upgrade triggers a re-baseline run on a stable checkpoint that is compared against the old pin before promotion. Silent runner upgrades are structurally impossible.

### Failure mode: an eval "passed" but nobody knows what it ran on

A CI job in the release pipeline invoked an eval two weeks ago. Someone is now trying to reproduce the score. The job log says `pytest ran ok`; the actual runner invocation was constructed from a mix of environment variables, a config file whose git ref is not recorded, and a shell script that curl'd a model endpoint whose route pointed at whichever weight file the serving stack was pinning at the time.

Plane mitigation: eval runs are first-class database rows with resolved artifact hashes, resolved model version, resolved runner version, and event log. Every CI job's eval invocation resolves through the plane and produces a `run_id`; the CI job's log stores the `run_id` and nothing else. Reproduction is `plane.rerun(run_id)`.

### Failure mode: two teams disagree because they picked different runners

The safety team's HarmBench score is 0.98 in their Inspect-based harness; the capability team's HarmBench score is 0.94 in an lm-eval-harness variant. Both teams have documentation, both are defensible. Neither can reconcile.

Plane mitigation: task registration nominates a canonical runner. A team that wants to run the "same" task under a different runner registers a *different task* — `harmbench_standard.inspect@1a2b3c4` vs. `harmbench_standard.lm_eval_harness@1a2b3c4` — so the divergence is named in the registry rather than hidden in configuration. The two scores are labeled separately and the warehouse never joins them silently.

## Guidance for the platform engineer

- **Adapters, not adapters-of-adapters.** Wrapping lm-eval-harness's Python API in a fake Inspect solver to "run everything through Inspect" is a very tempting architecture and it leaks abstraction bugs at every join. Give each runner its own first-class adapter.
- **The plane resolves; adapters execute.** Registry resolution happens at the plane before dispatch. Adapters see already-resolved artifacts; they cannot reach into the registry themselves.
- **The model-client layer belongs to the platform.** LiteLLM or a vendor gateway can be the substrate, but the version-pinning, quota, and lineage properties are the platform's, not the substrate's.
- **Normalized per-item results in the warehouse.** Aggregates are computed on top of per-item rows, not passed through from the runner. This is what makes cross-runner comparability structurally sound.
- **Dispatcher decisions are logged.** When a task has multiple compatible runners, the plane's choice for this run is written into the lineage. "Which runner did this land on" is a query, not an investigation.
- **Runner upgrades are re-baseline events.** The runner pin is a platform artifact; changes to it go through the same re-baseline discipline mod-110 Chapter 2 established for the incumbent-model swap.

## Summary

The multi-runner orchestration plane is what lets an organization use lm-evaluation-harness, Inspect, OpenAI evals, and internal harnesses without paying the O(N × M) cost of every team learning every runner. Users request evals by (task, model, config); the plane resolves registry references, picks a runner via a documented dispatch policy, invokes the runner through a small adapter contract, and normalizes results into a common per-item schema. Model plurality is handled symmetrically by a thin platform-owned model-client layer that enforces version-pinning, quota, and lineage properties regardless of the underlying gateway. The three failure modes the plane structurally prevents are silent runner-CLI drift, uncatalogued CI invocations that cannot be reproduced, and inter-team disagreement about "the same eval" because of hidden runner choices. The next chapter is about what happens when many of these evals want to run at once: platform-scope parallelism, cost controls, and judge-tier routing.
