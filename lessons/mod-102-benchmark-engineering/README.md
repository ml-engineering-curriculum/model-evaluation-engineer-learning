# mod-102-benchmark-engineering: Benchmark Engineering: Constructing, Versioning, and Decontaminating Datasets

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Source raw data with documented licensing and provenance and convert it into a benchmark-ready dataset
- Run a gold-standard labelling pipeline: instruction design, pilot, inter-annotator agreement, adjudication, gold rotation
- Detect benchmark contamination (n-gram overlap, embedding overlap, model-side log-likelihood probes) and produce a contamination report
- Version a benchmark (dataset hash, task definition hash, evaluator hash) and design a deprecation / rotation policy
- Manage public / private holdouts and the canary set used to detect post-hoc training-set inclusion

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
