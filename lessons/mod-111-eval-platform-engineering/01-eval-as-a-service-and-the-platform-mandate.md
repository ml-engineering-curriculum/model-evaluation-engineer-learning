# Eval-as-a-Service and the Platform Mandate

Every module before this one has treated evaluation as an *engagement* — an evaluator engineer picks up a task, wires together a dataset, a runner, a judge, a metric, and returns a report. That framing is correct for the individual evaluation and it is what mod-101 through mod-110 build the disciplines for. It is also what breaks the moment the organization is running more than one or two evaluations simultaneously across more than one or two model families.

At that point the failure modes are not about the evaluations themselves. They are about the platform underneath them: the same dataset is loaded three different ways by three teams and the results are silently incomparable; the same judge prompt exists in five slightly-diverged copies and no one can tell which version graded the report on the release manager's desk; the safety team's regression run and the research team's leaderboard sweep both saturate the vendor's rate limit at the same hour and every evaluation on the platform is late; a candidate that "passed CI" cannot be reproduced two weeks later because the exact model revision has been silently replaced by the vendor. Each of these is a *platform* problem, not an evaluation problem. This module builds the platform.

## What "eval-as-a-service" means in this module

"Eval-as-a-service" (EaaS) is the discipline of treating evaluation infrastructure the same way a mature organization treats data infrastructure, compute infrastructure, and observability infrastructure: as a shared, versioned, quota-controlled, SLO-owning platform that many product teams depend on and that a small platform team owns end-to-end. Concretely, an EaaS platform provides:

- **A single control plane** where a user (or a CI system) requests "run this eval on this model" without knowing which underlying runner will execute it, which region will host the workers, or which judge tier will grade the outputs.
- **A versioned registry** of the four kinds of eval artifacts — tasks, datasets, judges, and prompts — with content-addressed hashes so that "the same eval" and "a different eval" are unambiguous.
- **A pluggable execution layer** that runs the user's request through the appropriate runner (lm-evaluation-harness, Inspect, OpenAI evals, or an internal harness) without the user having to know the runner's CLI surface.
- **Platform-scope cost controls** — token budgets, judge-tier routing, priority queues — that are enforced by the platform, not policed by each team.
- **A results warehouse** with traceable lineage: every metric row can be traced back to the exact model, dataset, judge, prompt, decoding config, and seed that produced it, without hand-reconstruction.
- **Release-pipeline integrations and its own SLOs** — the eval platform is itself a service with uptime, latency, and correctness commitments to its consumers.

The delivered artifact of this module is the design of that platform. It is not a specific technology stack — the *right* stack depends on your organization's compute posture, tolerance for hosted-vendor lock-in, and the shape of your existing data and CI infrastructure — but the components are stable and the failure modes are predictable.

## Why a platform is the right unit of investment

Individual evaluations can and should be built without a platform when the organization is small. Three teams running two evals apiece do not need eval-as-a-service; they need a shared judge-prompt file, a shared dataset checkout, and enough discipline to log the model version alongside the metric. That is a legitimate stage in an organization's evolution and treating it as one is not evidence of technical debt.

The forcing functions that move an organization from "individual evaluations" to "eval as a service" are:

- **Multiplication of eval consumers.** Once a dozen teams each want to run their own regression suite, the O(N²) coordination cost of "please rebase your dataset onto the same fork we're using" or "please use the same judge revision so we can compare our numbers" is larger than the cost of the platform.
- **Multiplication of runners.** As soon as the organization is running lm-eval-harness for capability benchmarks, Inspect for red-team evals, an internal harness for RAG-with-tools, and a homegrown script for LLM-as-judge, the aggregate ops burden — credentials, retry logic, quota-tracking, results-parsing — is larger than the amortized cost of a runner-abstraction layer.
- **Multiplication of decision consumers.** When the release train, the safety review body, the fine-tuning experiment tracker, and the customer-facing benchmarks page all need to consume eval results, they need a warehouse. Point-to-point data flows between each eval producer and each consumer are the O(N × M) integration cost the warehouse pattern exists to eliminate.
- **Multiplication of cost surface.** A single team running one eval can eyeball the vendor bill at the end of the month. A dozen teams running dozens of evals across three vendors cannot; the cost has to be metered and attributed at the platform layer or the finance conversation becomes intractable.
- **Regulatory and audit pressure.** For organizations subject to the EU AI Act's general-purpose AI obligations (Article 55 for GPAI with systemic risk), the NIST AI Risk Management Framework's "Measure" function, or ISO/IEC 42001 information-management-system requirements, "we can reproduce any eval score that has been used in a public claim" is not optional. Ad-hoc infrastructure cannot deliver that; a platform with lineage can.

The through-line is that a platform is what makes the *nth* eval cheaper to run and easier to trust than the first. Every architectural choice in the rest of this module should be judged against that criterion.

## The five components (and how they map to this module's chapters)

The rest of the module builds an EaaS platform in five component layers. Each layer maps to a chapter and to one of the module's learning objectives.

- **Layer 1: the versioned registry (Chapter 2).** A content-addressed store of tasks, datasets, judges, and prompts. Answers the question "when someone says `mmlu@rev-3` or `helpfulness-judge-v1.2`, exactly which bytes do they mean?" and enforces immutability plus a deprecation lifecycle.
- **Layer 2: the multi-runner orchestration plane (Chapter 3).** An abstraction over the concrete runners (lm-eval-harness, Inspect, OpenAI evals, internal harnesses) that lets a user request an eval by (task, model, config) and routes the request to the appropriate runner without leaking the runner's CLI surface. Answers the question "how do I run 200 evals across five runners and one dashboard?"
- **Layer 3: platform-scope parallelism and cost controls (Chapter 4).** Token budgets, runner budgets, judge-tier routing, priority queues, per-tenant quotas, and rate-limit backoff. Answers the question "how does the platform stay useful when four teams try to saturate the same vendor at the same time?"
- **Layer 4: the eval data warehouse and lineage (Chapter 5).** The results schema, the hash-based lineage graph, the join keys, the retention policy. Answers the question "can I reproduce the score on the release-manager's desk in six months without hand-detective work?"
- **Layer 5: CI integration and platform SLOs (Chapter 6).** How the platform is called from release pipelines, how eval-blocking is wired into deploy tooling, and what the platform's own SLOs — availability, latency, correctness, cost predictability — look like. Answers the question "if the eval platform is down, does the release train stop or roll ahead?"

An organization does not need to build all five layers on day one, and it usually shouldn't. Chapter 2 is the load-bearing foundation — without a versioned registry, every other layer accumulates lineage debt that is expensive to pay back. Chapter 5 is the second most load-bearing — the warehouse pays for itself the first time someone asks "how did this metric look on this model in Q3." The other three layers can be built out incrementally once those two are solid.

## Where the eval platform sits relative to adjacent platforms

An eval platform does not exist in isolation. Four adjacent platforms deserve explicit disclaimers so the boundaries are clear:

- **The model-serving platform.** The eval platform *calls* a serving platform — a proxy in front of vLLM/SGLang/TGI, a hosted-vendor abstraction layer like LiteLLM, or a direct client library. It should not embed serving. Model rollout, autoscaling, and inference batching are the serving platform's job; the eval platform should be able to point at any of them via a stable interface.
- **The observability platform.** Modules 105 and 110 covered Arize Phoenix, Langfuse, and W&B Weave. The eval platform *emits* traces and evaluations into the observability platform. The eval platform is not the observability platform — trying to make it one couples the release-blocking eval discipline (which needs strong immutability, reproducibility, and audit) to the interactive-debug workflow (which needs latency, flexibility, and rapid iteration). Keep them separate.
- **The experiment-tracking platform.** MLflow, W&B, and internal experiment stores are where the fine-tuning and training teams live. The eval platform *consumes* model artifacts registered there and *emits* eval scores that can be attached to those artifacts. Some organizations put both eval and experiment tracking behind the same UI; that is a UI decision, not an architectural one.
- **The CI/CD platform.** GitHub Actions, GitLab CI, Buildkite, Argo Workflows. The eval platform *is called from* the CI/CD platform; it does not embed CI. Chapter 6 walks the integration pattern — the eval platform exposes a stable API (or CLI or webhook) that a release pipeline can invoke and consume.

An architecture diagram of a mature LLM organization has boxes for all five platforms and the arrows between them are the interface contracts this module makes explicit.

## Platform SLOs: what the eval platform owes its consumers

Once eval is a shared service, it acquires the same categories of commitment that any other shared platform acquires. Chapter 6 makes these concrete; the four categories are:

- **Availability.** The percentage of eval requests over a rolling window that return without infrastructure error (five-nines is not the right target here — most eval workloads are batch, not user-facing).
- **Latency.** The end-to-end wall-clock time from "user submits eval request" to "results in the warehouse and available in the UI." Distinguished at least at P50 and P95 and stratified by workload class (a 1M-token capability sweep and a 200-prompt regression suite have different SLOs).
- **Correctness.** The percentage of runs whose lineage is fully resolvable, whose datasets and judges match the requested revisions byte-for-byte, and whose results are numerically identical on a re-run under the same seed.
- **Cost predictability.** The percentage of workloads that complete within their declared token budget; the percentage of periods where the platform's aggregate vendor spend falls within its declared budget.

An eval platform without SLOs is an eval platform whose failures are invisible until a release-critical run has already been broken. Chapter 6 covers the shape of each SLO, how to derive its target, and how to instrument the platform to serve error budgets against them.

## A note on team shape and staged adoption

This module is structured as if you were building the platform greenfield. Most organizations do not — they inherit a spreadsheet of judge prompts and a Slack channel of dataset URLs and have to migrate. Two heuristics for staged adoption:

- **Migrate under one artifact type at a time.** Registries for judges (Chapter 2) tend to be the highest-leverage first migration because judge-prompt drift is where reproducibility fails first. Datasets are next. Tasks and full prompt templates can wait.
- **Migrate the runner abstraction after the registry.** A runner abstraction (Chapter 3) over unversioned tasks and datasets makes the runner's version-pinning story worse, not better. Fix the artifacts first, then unify the runners.

An eval platform is not a big-bang project; every layer this module builds should be shippable independently, and every organization's rollout order will differ.

## What this module does not cover

Two adjacent topics are close enough that they need explicit disclaimers.

- **The individual evaluation disciplines.** The construct-validity work of mod-101, the benchmark engineering of mod-102, the judge-calibration work of mod-105, the safety measurements of mod-109 — all of those are treated as *given inputs* to the platform in this module. The platform does not make an ill-formed evaluation well-formed; it makes a well-formed evaluation reproducible, comparable, and inexpensive. If the upstream evaluations are broken, a platform will scale the breakage.
- **Full systems-design integration.** Mod-112 is the capstone that wires an eval program (the platform this module builds, plus the disciplines from every earlier module) into a launch scorecard, a research feedback loop, and an external-reporting story. This module builds the platform; mod-112 shows how a whole organization uses it end-to-end.

## Summary

An eval-as-a-service platform is what turns individual evaluation engagements into a shared, versioned, budgeted, SLO-owning service that many teams depend on. The forcing functions are multiplication — of eval consumers, runners, decision consumers, cost surface, and audit obligations — and the payoff is that the *nth* eval becomes cheaper and more trustworthy than the first. The platform is built in five layers: a versioned registry of tasks, datasets, judges, and prompts; a multi-runner orchestration plane; platform-scope cost and parallelism controls; a results warehouse with hash-based lineage; and CI integration with the platform's own SLOs. The eval platform is not a serving platform, an observability platform, an experiment tracker, or a CI system; it interfaces with each and stays inside its own boundaries. The next chapter builds the load-bearing foundation: the versioned registry that makes every downstream layer's lineage story possible.
