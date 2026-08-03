# From Raw Data to a Benchmark-Ready Dataset

Chapter 1 stopped when a raw item had a license, a provenance record, and a hash. This chapter turns that raw store into a benchmark: a versioned, schema-conformant set of items, with a frozen task definition, a scoring rule, and reproducible splits. Downstream chapters version, decontaminate, and rotate what you build here. Get the schema wrong and every downstream chapter has to work around the wrong thing.

## What a benchmark-ready dataset is (and is not)

A benchmark-ready dataset is a set of `(input, reference, metadata)` records with a fixed schema, plus a task specification that says how the model is prompted, how outputs are scored, and how per-item scores aggregate. The dataset without the task spec is a data dump; the task spec without the dataset is a rubric. The pair, jointly hashed and versioned, is a benchmark.

Concretely, a v1 release contains:

- **`data/`** — one file per split (`train`, `dev`, `public_test`, `private_test`, `canary`) in Parquet or JSONL, conforming to the schema below.
- **`task.yaml`** — the task specification: prompt template, allowed model outputs, scoring function reference, aggregation rule.
- **`evaluator.py` (or a pointer to a pinned version)** — the code that produces per-item scores from `(reference, output)`.
- **`splits.yaml`** — the split definition (which item IDs go in which split, and the split-assignment seed).
- **`MANIFEST.json`** — the dataset hash, the task hash, the evaluator hash, the release version, and the counts per split. Chapter 6 covers this file in detail.
- **`ATTRIBUTION.md`** — the attribution manifest for items whose license requires it.
- **`README.md`** — human-facing description: construct claim, intended use, known limitations, how to cite.

If any of these is missing, the artifact is not shippable; there is no place downstream to add it that would not undermine reproducibility.

## The per-item schema

Fix these columns in a stable order and let application-specific fields sit alongside them. The core is small on purpose — you want new evals to reuse most of the schema.

- **`item_id`** — a stable, opaque string. Not a hash of the content (that changes when you fix a typo). Not the row index (that changes on resplit). Generate a UUIDv4 or a monotonic ID at ingest time and never reuse it. Item IDs are the primary key for every downstream operation: contamination reports, canary lookups, deprecation lists, per-item score joins.
- **`input`** — the model-facing input. For text tasks: a string. For multimodal: a structured record with file URIs, mime types, and content hashes. Do not put the prompt template here; that lives in `task.yaml`. `input` is the *substance*, not the wrapping.
- **`reference`** — the gold reference. For classification: the class label. For extraction: the extracted span or structured object. For generation: either an accepted-answer set (`{"aliases": [...]}`), a rubric-with-graders, or a reference-free spec if the evaluator does not need one.
- **`metadata`** — a nested object with subfields you can slice on. Reserve a slot for at minimum: `provenance` (the license/source record from Chapter 1), `domain`, `difficulty`, `length`, `language`, `pii_status`, `created_at`.
- **`split`** — one of the split names. Included in the row for join safety even though `splits.yaml` is the authority; the redundancy catches leakage.

Two schema commandments you will pay for if you violate them.

**Do not put the prompt template in the item.** If every row has `"question": "…"`, `"prompt_prefix": "Answer this question: "`, `"prompt_suffix": "\n\nAnswer:"`, then changing the template requires rewriting every row — and, worse, you can no longer tell whether a score change came from the model or from a template edit. Store `question` in `input`; put the prefix and suffix in `task.yaml`, versioned separately.

**Do not put the scoring rule in the item.** "Accept any of these substrings" is scoring policy, not data. Put the acceptance set in `reference`; keep the "match substring case-insensitively after normalization" logic in `evaluator.py`. The evaluator is versioned so score deltas from a scorer bug fix are attributable.

## Task specification: the frozen contract

`task.yaml` is short. It has to be, because it will be argued over and edited more than the data will:

```yaml
name: internal-support-triage
version: 1.0.0
input_field: input
reference_field: reference
prompt_template: |
  You are triaging a support ticket. Assign one label from the set below.
  Ticket: {{ input }}
  Label:
allowed_outputs:
  regex: ^(billing|technical|abuse|other)$
scoring:
  function: exact_match_after_normalize
  normalize:
    lowercase: true
    strip_whitespace: true
aggregation:
  primary: accuracy
  slices:
    - metadata.domain
    - metadata.language
decoding:
  temperature: 0.0
  max_tokens: 8
  stop: ["\n"]
```

Freeze this file at release time. Any edit is a version bump (Chapter 6), because it changes what the benchmark measures. In particular:

- **`prompt_template`** is part of the construct definition (see mod-101 Chapter 2 on construct-irrelevant variance). Move a word, get a different number.
- **`allowed_outputs`** decides whether a model that hedges with prose gets zero or gets a partial parse. Freeze the parser rule alongside the regex.
- **`decoding`** is the reproducibility knob for stochastic models. Pin temperature, max tokens, and stop sequences. If the benchmark specifically wants to sample multiple times (pass@k), state `k` and the sampling seed here.

For evals that permit multiple prompt templates (e.g. HELM-style multi-prompt averaging), the template *set* is what's frozen. The set has a hash; edits to it are version bumps.

## The evaluator: code, hashed, pinned

`evaluator.py` implements the scoring function referenced in `task.yaml`. Two rules:

1. **Deterministic.** Given `(reference, output, item)`, the score is a pure function of the inputs and the evaluator version. No calls to a live LLM judge inside a scorer marked deterministic. If your scorer is a judge, mark it stochastic and pin the judge model version explicitly (mod-105).
2. **Version-hashed.** The evaluator file (or the specific graders it imports) has a semantic version. That version participates in the benchmark manifest hash (Chapter 6). A scorer bug fix that reports "we found we were counting a common answer as wrong" moves the number for every model on every past run — you want that reflected as an evaluator version bump, not silently.

For a benchmark that ships an LLM-as-judge scorer, `evaluator.py` still contains the deterministic parts (prompt template, parsing of the judge output, aggregation) and pins the judge model as a config; the judge's own weights are the remaining source of non-determinism, tracked as a mod-105 concern.

## Splits: `train`, `dev`, `public_test`, `private_test`, `canary`

Public benchmarks in ML typically ship `train / dev / test`. For eval-engineered benchmarks — where the primary use is measuring models, not training them — the more useful partition is:

- **`train` (optional).** Included if the benchmark also functions as a training set. For an eval-first benchmark, this is usually empty.
- **`dev` (small, published).** The one models can be tuned against without a leakage claim. Sized to give a signal (a few hundred items) but not to be the reporting number.
- **`public_test`.** The one reported on leaderboards. Assume it will end up in future pretraining corpora. Chapter 5 explains how to detect that; Chapter 7 explains how to keep something in reserve when it does.
- **`private_test`.** Held privately; scored by submitting model outputs (or a model API) to the benchmark maintainer. Not published. Used to detect divergence between public and private performance — a divergence is a leakage signal.
- **`canary`.** A small set of items with distinguishing text designed to be greppable in future training corpora. Chapter 7 covers design.

The split assignment is deterministic and reproducible. Store it in `splits.yaml`:

```yaml
seed: 20260803
strategy: stratified_by_domain
counts:
  dev: 500
  public_test: 2000
  private_test: 2000
  canary: 100
```

The `splits.yaml` plus the `data/` files plus the seed reproduce the exact split; save a `split_hash` in the manifest so a reader can verify.

Two split-design details that matter more than they look.

**Split unit vs. row unit.** If your data has natural groupings — dialogues, users, source URLs, question templates — assign at the group level, not the row level. Same-user rows in both dev and test leak the user; same-template rows leak the template. Group split assignment is the single-line change that prevents an entire class of subtle leakage.

**Strata for representativeness.** If you have language, domain, or difficulty subgroups, stratify. Random splits at small `n` can drop a subgroup out of `public_test` entirely; you find out only when the slice score is undefined.

## Cleaning: what to do to the raw item, and what not to

Before an item enters `data/`, run a minimal, documented cleaning pipeline. The passes worth naming:

- **Encoding normalization.** UTF-8, NFC-normalized. Silently mixed encodings break tokenizers and scorers in ways that produce quiet wrong answers.
- **Deduplication.** Exact-hash duplicates within the raw store are removed. Near-duplicates (MinHash or embedding-clustered) are flagged for review; usually only one representative survives. Duplicates inflate reported scores in proportion to their multiplicity.
- **PII scrubbing.** For any raw source that could contain personal data (user prompts, support tickets, forum posts), run a named-entity redactor plus a regex sweep for the obvious formats (emails, phones, SSNs, credit cards, IPs). Record `pii_status` accordingly. Publish the scrubber version so a downstream reviewer can trace what was redacted.
- **Language identification and filtering.** If the benchmark is single-language, run a language ID model and drop out-of-language items. If it's multi-language, tag each item.
- **Length filtering.** Drop items that are too short to be scorable or too long for your target context window. Document the thresholds; length filtering is a construct-shaping choice.
- **Explicit-content filtering.** If items are scraped, run a toxicity classifier and either drop or gate; consult your organization's policy for whether toxic items are needed for a safety eval (see mod-109).

What *not* to do to the raw item:

- **Do not fix typos in the item unless the task is orthography.** A typo is signal about real inputs. Silent fixes bias the benchmark toward clean input.
- **Do not rewrite the item for clarity.** Rewriting turns a naturally-sourced item into a synthesized one, which is a different eval.
- **Do not merge items.** Merging destroys `item_id` uniqueness and breaks every downstream join.

Every cleaning step is a `data_cleaning_step_id` recorded in the item's metadata so a downstream user can inspect which passes touched a row.

## PII, consent, and the "cannot ship" flag

For items sourced from user data, the schema needs an escape hatch: sometimes an item is technically clean but has to be dropped for consent or contractual reasons. Add a `blocklist_reason` field. When populated, the item never enters a shipped split; it stays in the raw store with an explanation. Chapter 6 explains how the blocklist evolves — the blocklist is versioned along with the data.

## Reproducibility: the "regenerate the dataset from scratch" test

The test is simple. Given the raw store, the ingest pipeline version, and the seeds in `task.yaml` and `splits.yaml`, running the pipeline reproduces `data/`, `task.yaml`, `evaluator.py`, and `MANIFEST.json` byte-identically. If it does not, one of these is likely the cause:

- The cleaning pipeline reads from a live external service (a language ID API that has drifted).
- The scrubber pulls a model whose weights are not pinned.
- Split assignment uses `random` instead of a seeded generator.
- The item ordering depends on filesystem enumeration order rather than a sorted key.

Fix each of these at authoring time. The end goal is that Chapter 6's `dataset_hash` is stable across environments; the "regenerate" test is the assertion that the hash is real.

## A minimal end-to-end walk

Suppose you are building an internal support-triage eval from support tickets. The end-to-end shape:

1. Sourcing (Chapter 1): pull tickets from the ticketing system with the license/provenance record. Consent status: `broad-tos-consent`; commercial use allowed; attribution not required.
2. Cleaning: NFC-normalize, dedupe by content hash, scrub PII, tag language, drop items shorter than 5 tokens or longer than 2048.
3. Labelling (Chapters 3–4): three annotators per item, adjudication, gold rotation.
4. Splitting: `splits.yaml` with `dev=500, public_test=2000, private_test=2000, canary=100`, stratified by `metadata.domain`, seeded.
5. Task spec: `task.yaml` with a single fixed prompt template, exact-match scoring against the gold label, per-domain slice reporting.
6. Evaluator: `evaluator.py` implements exact match with normalization; version 1.0.0.
7. Manifest: `MANIFEST.json` with the dataset hash, task hash, evaluator hash, split hash (Chapter 6).
8. Release: tag `v1.0.0` in the benchmark repo; publish `dev` and `public_test`; keep `private_test` and `canary` in the private artifact store.

Every step corresponds to a chapter in this module. If any one step is missing, the release is not shippable.

## Summary

A benchmark-ready dataset is a schema-conformant set of `(input, reference, metadata)` items, plus a frozen `task.yaml` (prompt, output constraints, scoring, aggregation, decoding), plus a pinned `evaluator.py`, plus reproducible `splits.yaml`, plus a manifest of hashes. Keep the schema small and stable. Split at the natural group unit, not the row unit. Do minimal cleaning, document each pass, and never silently rewrite items. Enforce reproducibility by requiring that the pipeline regenerates every artifact byte-identically from raw data plus seeds. Chapter 3 picks up the labelling pipeline — how the `reference` field gets its values.
