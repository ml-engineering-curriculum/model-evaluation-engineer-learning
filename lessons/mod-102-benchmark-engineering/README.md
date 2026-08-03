# mod-102-benchmark-engineering: Benchmark Engineering: Constructing, Versioning, and Decontaminating Datasets

**Estimated effort:** 16 hours

Benchmark engineering is the discipline of turning raw data into a reproducible measurement instrument you can defend under audit and keep meaningful under adversarial pressure. This module walks the full lifecycle: sourcing raw data with a documented license and provenance chain, converting it into a schema-conformant task-scoped dataset, running the gold-standard labelling pipeline that makes the reference labels credible, detecting benchmark contamination with three complementary detector families, versioning the artifact so score comparisons stay honest across time, and standing up the public/private and canary infrastructure that turns future contamination from an inference into an observation.

Every chapter connects to the failure modes mod-101 identified — construct validity, internal validity, and estimation uncertainty — and gives the operational answer for the benchmark-authoring role rather than the reader-of-benchmarks role.

## Learning objectives

- Source raw data with documented licensing and provenance and convert it into a benchmark-ready dataset.
- Run a gold-standard labelling pipeline: instruction design, pilot, inter-annotator agreement, adjudication, gold rotation.
- Detect benchmark contamination (n-gram overlap, embedding overlap, model-side log-likelihood probes) and produce a contamination report.
- Version a benchmark (dataset hash, task definition hash, evaluator hash) and design a deprecation / rotation policy.
- Manage public / private holdouts and the canary set used to detect post-hoc training-set inclusion.

## Lecture chapters

1. [`01-sourcing-data-license-and-provenance.md`](01-sourcing-data-license-and-provenance.md) — per-item provenance schema, SPDX-tagged licensing, and the source-dossier pattern.
2. [`02-from-raw-data-to-benchmark-schema.md`](02-from-raw-data-to-benchmark-schema.md) — dataset schema, `task.yaml` and pinned evaluators, split design, and the reproducibility test.
3. [`03-annotation-instruction-design-and-pilots.md`](03-annotation-instruction-design-and-pilots.md) — instruction structure, decision rules vs. judgment, the pilot round, and calibration.
4. [`04-inter-annotator-agreement-and-adjudication.md`](04-inter-annotator-agreement-and-adjudication.md) — Cohen's/Fleiss' κ and Krippendorff's α, adjudication protocols, and gold rotation.
5. [`05-detecting-benchmark-contamination.md`](05-detecting-benchmark-contamination.md) — surface n-gram, embedding overlap, and model-behavioral probes; the contamination report artifact.
6. [`06-versioning-benchmarks-and-deprecation.md`](06-versioning-benchmarks-and-deprecation.md) — dataset/task/evaluator hashes, semver for benchmarks, delta reports, and deprecation lifecycle.
7. [`07-holdouts-and-canary-sets.md`](07-holdouts-and-canary-sets.md) — public/private test splits, canary design, and operating both under real access pressure.

## Exercises

Five hands-on prompts under [`exercises/`](exercises/). Each is self-contained and can be completed after finishing the chapters it depends on.

- [`exercise-01-license-and-provenance-trace.md`](exercises/exercise-01-license-and-provenance-trace.md) — build the per-item provenance record for a real corpus and a defensible source dossier.
- [`exercise-02-gold-set-with-iaa-and-adjudication.md`](exercises/exercise-02-gold-set-with-iaa-and-adjudication.md) — run a three-annotator pilot, compute κ, adjudicate the residual, and write an IAA report.
- [`exercise-03-contamination-detection-pipeline.md`](exercises/exercise-03-contamination-detection-pipeline.md) — build a three-detector contamination pipeline and produce a report for a real benchmark.
- [`exercise-04-benchmark-versioning-and-deprecation-policy.md`](exercises/exercise-04-benchmark-versioning-and-deprecation-policy.md) — implement the three-hash manifest, run a v1.0→v1.1 rotation, and write a deprecation policy.
- [`exercise-05-canary-set-design.md`](exercises/exercise-05-canary-set-design.md) — design public and private canary items, register them, and specify the periodic probe.

Reference solutions live in the paired [`model-evaluation-engineer-solutions`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-solutions) repository.

## Labs and quizzes

- [`labs/`](labs/) — long-form hands-on labs (scaffold in place; content authored in a subsequent cycle).
- [`quizzes/`](quizzes/) — knowledge checks (scaffold in place; content authored in a subsequent cycle).

## Resources

See [`resources.md`](resources.md) for primary references — SPDX License List, Datasheets for Datasets (Gebru et al. 2018), Cohen 1960, Fleiss 1971, Krippendorff 2004, GPT-3 (Brown et al. 2020) and PaLM (Chowdhery et al. 2022) contamination sections, Sainz et al. 2023, Golchin & Surdeanu 2023, Oren et al. 2023, BIG-bench (Srivastava et al. 2022), and the standard versioning and dataset-management tooling references.
