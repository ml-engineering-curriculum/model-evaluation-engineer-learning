# Resources for mod-111-eval-platform-engineering (Eval Platform Engineering: Eval-as-a-Service)

Primary references for the material in this module. Prefer the original paper, the maintained tool repository, or the official documentation over secondary summaries.

## Eval runners (Chapter 3)

- **EleutherAI `lm-evaluation-harness`.** The reference implementation for the Language Model Evaluation Harness ecosystem. Task YAML format, `TaskConfig` schema, and adapter surface used across the community. Repository: [`EleutherAI/lm-evaluation-harness`](https://github.com/EleutherAI/lm-evaluation-harness). See [`docs/task_guide.md`](https://github.com/EleutherAI/lm-evaluation-harness/blob/main/docs/task_guide.md) for the task-registration surface.
- **UK AI Safety Institute `Inspect`.** Solver / scorer framework for evals with tool use and multi-step scoring. Docs: [inspect.aisi.org.uk](https://inspect.aisi.org.uk/), including [scorers](https://inspect.aisi.org.uk/scorers.html), [datasets](https://inspect.aisi.org.uk/datasets.html), and [`model_graded_qa`](https://inspect.aisi.org.uk/scorers.html#model-graded-qa). Repository: [`UKGovernmentBEIS/inspect_ai`](https://github.com/UKGovernmentBEIS/inspect_ai).
- **OpenAI `evals`.** The `modelgraded` spec and `cot_classify` scoring pattern; broad task catalogue. Repository: [`openai/evals`](https://github.com/openai/evals). Completion-fn docs: [`docs/completion-fns.md`](https://github.com/openai/evals/blob/main/docs/completion-fns.md).
- **Stanford CRFM `HELM`.** Multi-metric benchmark framework with a documented adapter, metric, and scenario abstraction. Repository: [`stanford-crfm/helm`](https://github.com/stanford-crfm/helm). Docs: [crfm-helm.readthedocs.io](https://crfm-helm.readthedocs.io/).
- **Liang, P., Bommasani, R., et al. (2022).** "Holistic Evaluation of Language Models" (HELM). The methodology paper HELM is built from — worth reading as an example of a defensible multi-metric eval framework. [arXiv:2211.09110](https://arxiv.org/abs/2211.09110).

## Judges and judge platforms (Chapter 2, Chapter 4)

- **Zheng, L., Chiang, W.-L., et al. (2023).** "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena." *NeurIPS Datasets & Benchmarks*. The reference paper for the modern LLM-as-judge setup, judge biases, and Arena-style ELO. [arXiv:2306.05685](https://arxiv.org/abs/2306.05685).
- **Kim, S., et al. (2024).** "Prometheus 2: An Open Source Language Model Specialized in Evaluating Other Language Models." *EMNLP*. The reference open-weights dedicated judge for the "Tier 1" role of Chapter 4's judge tiering. [arXiv:2405.01535](https://arxiv.org/abs/2405.01535). Repository: [`prometheus-eval/prometheus-eval`](https://github.com/prometheus-eval/prometheus-eval).
- **Verga, P., et al. (2024).** "Replacing Judges with Juries: Evaluating LLM Generations with a Panel of Diverse Models." Motivates the ensemble-of-cheaper-judges pattern that composes with judge-tier routing. [arXiv:2404.18796](https://arxiv.org/abs/2404.18796).

## Model-client and gateway layer (Chapter 3, Chapter 4)

- **BerriAI `LiteLLM`.** Unified client wrapper across hosted vendors with rate-limit and retry primitives. Repository: [`BerriAI/litellm`](https://github.com/BerriAI/litellm). Docs: [docs.litellm.ai](https://docs.litellm.ai/).
- **`Portkey`.** Commercial gateway with routing, retries, and observability across vendors. Repository / docs: [portkey.ai](https://portkey.ai/) and [`Portkey-AI/gateway`](https://github.com/Portkey-AI/gateway).
- **Anthropic prompt caching.** Official documentation for cacheable prefix hints on the Anthropic API: [docs.claude.com/en/docs/build-with-claude/prompt-caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching). See also the pricing implications section for the discount rate on cache hits.
- **OpenAI prompt caching.** Automatic caching on long prompts on OpenAI's API: [platform.openai.com/docs/guides/prompt-caching](https://platform.openai.com/docs/guides/prompt-caching).
- **Google Gemini context caching.** Explicit context-cache primitive for the Gemini API: [ai.google.dev/gemini-api/docs/caching](https://ai.google.dev/gemini-api/docs/caching).
- **Anthropic Message Batches API.** Batch-pricing API with a 24-hour SLA — the batch pattern from Chapter 4: [docs.claude.com/en/docs/build-with-claude/batch-processing](https://docs.claude.com/en/docs/build-with-claude/batch-processing).
- **OpenAI Batch API.** Batch endpoint with 24-hour SLA and ~50% discount: [platform.openai.com/docs/guides/batch](https://platform.openai.com/docs/guides/batch).

## Registry, artifact stores, and content addressing (Chapter 2, Chapter 5)

- **Hugging Face Hub revisions.** Revision SHA behavior on `datasets` and `models`: [huggingface.co/docs/hub/repositories-revisions](https://huggingface.co/docs/hub/repositories-revisions). The reference for dataset immutability by revision.
- **Weights & Biases Artifacts.** Versioned artifact store with content-addressed digests and lineage graph: [docs.wandb.ai/guides/artifacts](https://docs.wandb.ai/guides/artifacts).
- **MLflow Model Registry.** Model versioning, aliases, and stage transitions: [mlflow.org/docs/latest/model-registry.html](https://mlflow.org/docs/latest/model-registry.html).
- **OCI image manifest and content-addressed distribution.** The container-image ecosystem's approach to immutable content-addressed artifacts; a good reference for the content-hash mechanics used in this chapter. [Open Containers Distribution Spec](https://github.com/opencontainers/distribution-spec).
- **Sigstore `cosign`.** Signature and provenance for artifacts — relevant when the registry hosts artifacts consumed under regulatory audit. [github.com/sigstore/cosign](https://github.com/sigstore/cosign).

## Data warehouse substrates (Chapter 5)

- **Apache Iceberg.** Table format for large analytic tables with schema evolution, time-travel, and hidden partitioning. Docs: [iceberg.apache.org](https://iceberg.apache.org/). The reference lakehouse table format for a warehouse-shape substrate.
- **Delta Lake.** Alternative lakehouse table format from Databricks. Docs: [delta.io](https://delta.io/).
- **Apache Parquet.** Column store format used by lakehouse table formats. Spec: [parquet.apache.org](https://parquet.apache.org/).
- **DuckDB.** In-process analytical database useful for eval warehouses at the O(10⁸) row scale. Docs: [duckdb.org/docs](https://duckdb.org/docs/).
- **Trino.** SQL query engine used against Iceberg / Delta / Hive-metastore-backed tables. Docs: [trino.io/docs](https://trino.io/docs/current/).

## Observability platforms (Chapter 3, Chapter 5, Chapter 6)

- **Arize `Phoenix`.** Open-source LLM observability with deep OpenInference integration. Docs: [docs.arize.com/phoenix](https://docs.arize.com/phoenix). Repository: [`Arize-ai/phoenix`](https://github.com/Arize-ai/phoenix).
- **Langfuse.** Open-source LLM observability with a Score primitive and prompt-management surface. Docs: [langfuse.com/docs](https://langfuse.com/docs). Repository: [`langfuse/langfuse`](https://github.com/langfuse/langfuse).
- **Weights & Biases `Weave`.** W&B's LLM observability layer, with the `weave.op` decorator model. Docs: [weave-docs.wandb.ai](https://weave-docs.wandb.ai/).
- **`OpenInference`.** OpenTelemetry-family convention for LLM spans, maintained by Arize. Repository: [`Arize-ai/openinference`](https://github.com/Arize-ai/openinference). Spec: [github.com/Arize-ai/openinference/tree/main/spec](https://github.com/Arize-ai/openinference/tree/main/spec).
- **`OpenTelemetry`.** The upstream tracing / metrics / logs spec that OpenInference extends. Docs: [opentelemetry.io/docs](https://opentelemetry.io/docs/).

## SRE, SLOs, and error budgets (Chapter 6)

- **Beyer, B., Jones, C., Petoff, J., & Murphy, N. R. (2016).** *Site Reliability Engineering.* O'Reilly. The canonical text on SLOs, error budgets, and the operational discipline this chapter borrows from. Full text online: [sre.google/sre-book/table-of-contents/](https://sre.google/sre-book/table-of-contents/). Chapter 4 ("Service Level Objectives") is the direct reference for the SLO framing.
- **Beyer, B., Murphy, N. R., Rensin, D. K., Kawahara, K., & Thorne, S. (Eds.) (2018).** *The Site Reliability Workbook.* O'Reilly. The applied companion; Chapters 2–4 walk SLO derivation from historical distributions. [sre.google/workbook/table-of-contents/](https://sre.google/workbook/table-of-contents/).
- **Google Cloud SRE resource: "Error budget policy."** Reference document for wiring the error-budget mechanic into an on-call rotation: [sre.google/workbook/error-budget-policy/](https://sre.google/workbook/error-budget-policy/).

## Standards and regulatory context (all chapters)

- **NIST AI Risk Management Framework (AI RMF 1.0), 2023.** The "Measure" function is the direct interface with evaluation engineering; the platform this module builds is the primary artifact organizations use to satisfy Measure at scale. [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework).
- **NIST AI RMF Generative AI Profile (NIST AI 600-1), 2024.** GAI-specific extension of the RMF with concrete measurement expectations for generative systems. [doi.org/10.6028/NIST.AI.600-1](https://doi.org/10.6028/NIST.AI.600-1).
- **EU AI Act (Regulation (EU) 2024/1689).** Obligations for high-risk systems and general-purpose AI models with systemic risk; Article 55 sets the eval and post-market monitoring obligations that a platform of this shape supports. [EUR-Lex text](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).
- **ISO/IEC 42001:2023, Information technology — AI management system.** The AI-specific management-system standard; the platform's SLO and change-management disciplines map to its Clause 9 (performance evaluation) and Clause 10 (improvement) sections. [ISO catalogue entry](https://www.iso.org/standard/81230.html).
- **ISO/IEC 25059:2023, SQuaRE — Quality model for AI systems.** AI-specific quality model that the platform's metric taxonomy composes with. [ISO catalogue entry](https://www.iso.org/standard/80655.html).

## Data privacy and secrets scanning (Chapter 5)

- **Yelp `detect-secrets`.** Repository-scale secret-scanning heuristics; useful as the payload-column scanner. Repository: [`Yelp/detect-secrets`](https://github.com/Yelp/detect-secrets).
- **TruffleHog OSS.** Credential detection with verifier plugins; the ingest-time quarantine layer of Chapter 5 can be built on it. Repository: [`trufflesecurity/trufflehog`](https://github.com/trufflesecurity/trufflehog).
- **NIST SP 800-88 Rev. 1, Guidelines for Media Sanitization (2014).** The reference for the "drop the key, drop the data" retention pattern used with column-level encryption. [csrc.nist.gov/publications/detail/sp/800-88/rev-1/final](https://csrc.nist.gov/publications/detail/sp/800-88/rev-1/final).

## Adjacent module cross-references

- **mod-101 (evaluation foundations).** Bootstrap confidence intervals, paired comparisons, and FDR — the statistical machinery Chapter 5's `run_aggregate_metric` schema stores and Chapter 6's SLO derivation applies. See in particular mod-101 Chapter 4 (bootstrap) and Chapter 6 (multiple comparisons).
- **mod-105 (LLM-as-judge platforms).** Judge calibration, agreement statistics, and biases — the discipline behind every judge revision Chapter 2's registry stores.
- **mod-109 (safety and red-team eval).** Safety-eval measurements — the workload that Chapter 4's `safety-canary` class runs and that Chapter 2's registry hosts as separately-governed artifacts.
- **mod-110 (production eval and regression).** The offline regression suite, shadow, A/B, and continuous monitoring altitudes that this module's platform serves.
- **mod-112 (eval systems design).** The systems-design capstone that composes this platform with the disciplines from every earlier module into an end-to-end launch scorecard.
