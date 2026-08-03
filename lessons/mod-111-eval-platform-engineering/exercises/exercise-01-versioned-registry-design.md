# exercise-01: Versioned Registry Design

**Estimated effort:** 3 hours

## Objective

Design and implement a working versioned registry for the four eval artifact types from Chapter 2 — tasks, datasets, judges, and prompts — with content-addressed immutable revisions, schema-enforced registration, semver conventions, a documented deprecation lifecycle, and a small CLI that exercises each. The deliverable is the registry itself (schema + implementation) plus a written note that shows, on concrete examples, how the registry structurally prevents the three failure modes Chapter 2 named (silent dataset divergence, judge drift under hosted vendors, unversioned rubric edits).

## Prerequisites

- mod-111 Chapter 2 (versioned registry). Chapter 1 for framing and Chapter 5 for the downstream schema the registry's foreign keys resolve into.
- mod-105 (LLM-as-judge platforms) — enough understanding of judge configurations that "a judge revision pins its backend model version" is not an abstract requirement.
- mod-101 Chapter 1 (validity) — the discipline that motivates why "same benchmark, different bytes" is a real failure and not a hypothetical.
- Python 3.11+ with `sqlalchemy` (or `sqlmodel`), `pydantic>=2`, `pyyaml`, `click` (or `typer`), a local Postgres or SQLite database, and enough object-store scaffolding to store the registered artifact bytes (a directory-backed fake is fine for the exercise).

## Requirements

### Part A — the data model

Implement the schema from Chapter 2 with at minimum these tables (SQLAlchemy models, migrations included):

- `artifact(id, kind, name, owner_team, created_at)` where `kind ∈ {task, dataset, judge, prompt}`.
- `artifact_revision(id, artifact_id, semver, content_hash, storage_uri, metadata jsonb, created_at, created_by, deprecated_at, deprecation_note, successor_revision_id)` with `UNIQUE(artifact_id, content_hash)`.
- `artifact_reference(from_revision_id, to_revision_id, reference_type)` where `reference_type ∈ {uses_dataset, uses_prompt, uses_judge}`.

Content hashes are `sha256` of a **canonicalized** serialization — JSON with sorted keys, LF line endings, no trailing whitespace. Formatting-only changes must not produce a new hash; content changes must.

### Part B — kind-specific schema validators

Ship a per-kind validator in `registry/validators/{task,dataset,judge,prompt}.py`. At minimum:

- **Dataset validator** rejects registrations without `license`, `source_url`, `row_count`, `schema_definition` in the metadata blob.
- **Judge validator** rejects LLM-judge registrations without a versioned `backend_model` (a bare `gpt-4` route fails; `anthropic:claude-sonnet-4-6@2026-02-01` passes). Requires a `rubric_prompt_ref` field that resolves to a registered prompt revision. Requires a `calibration_kappa` field with a documented calibration-set reference (empty is allowed; unset is not).
- **Prompt validator** parses the template with the declared engine, extracts the placeholder variables, and rejects registrations whose declared `input_variables` do not match the template placeholders exactly.
- **Task validator** requires a registered `dataset_ref`, `prompt_ref`, `metric.name`, `metric.version`, `decoding_config`, and a non-empty `runner_compatibility` list.

Every validation failure returns a machine-readable error code and a human-readable message; the CLI surfaces both.

### Part C — the CLI

Ship `registry.cli` with at minimum:

- `registry register <kind> --file=<path> [--semver=<x.y.z>]` — validates, computes the content hash, uploads the bytes to the fake object store, writes the `artifact_revision` row. Rejects re-registration of an existing content hash (idempotent no-op with a clear message).
- `registry list <kind> [--name=<name>] [--include-deprecated]` — lists artifacts and their revisions.
- `registry show <kind>/<name>@<semver-or-hash>` — resolves a reference and prints the full lineage (revision fields, artifact references, deprecation state).
- `registry deprecate <kind>/<name>@<semver> --successor=<kind>/<name>@<semver> --note=<text>` — marks the revision `sunset`, populates the successor pointer. Requires the successor to already exist.
- `registry archive <kind>/<name>@<semver>` — marks the revision `archived` (still queryable, no longer executable). Requires the revision to have been `sunset` for at least a documented cooldown (e.g., 30 days in production; overridable in the exercise for testing).

### Part D — semver and re-baseline mechanics

Implement a `registry rebaseline-check` command that, given a proposed revision and its predecessor, checks the semver claim against a fixture:

- A **patch** bump is accepted only if a diff of the canonicalized bytes shows only documentation / whitespace changes (a comment-only edit is patch; a rubric-anchor rewording is not).
- A **minor** bump is accepted for backwards-compatible additions (a new slice on a dataset whose original slices are unchanged; a new optional field on a task).
- A **major** bump is accepted for anything that changes the metric semantics (a new metric definition, a new few-shot policy, a substantive rubric edit).

Ambiguous edits — the ones where the diff crosses the "affects the metric" line but the author isn't sure — are treated as major bumps by default. The command emits a warning and requires an explicit `--force-minor` or `--force-patch` with a documented reason.

### Part E — the three-failure-mode demonstration

Write a `FAILURE_MODES.md` alongside the registry that demonstrates, with runnable commands, how the registry prevents each of the three failure modes from Chapter 2. Each demonstration is short (10–30 lines of shell + registry commands + expected output) and shows:

1. **Silent dataset divergence.** Register the same benchmark from two different sources; show that the registry names them separately and that a task pointing at one cannot silently be run against the other.
2. **Judge drift under hosted vendors.** Attempt to register a judge with an unversioned `gpt-4` route; show the rejection. Register a versioned judge revision and show that a vendor "roll" surfaces as a *new* judge revision with its own content hash rather than a silent change to the existing one.
3. **Unversioned rubric edits.** Edit an existing rubric prompt in place, attempt to register it under the same semver as its predecessor, and show the rejection. Register it as a new revision, run the `rebaseline-check`, and show the semver classification.

## Starter guidance

- **Start with the schema and the content-hash canonicalization, not the CLI.** Every other piece depends on the canonicalization being right; formatting-only diffs producing different hashes is the failure mode a whole afternoon of debugging goes into.
- **Use pydantic models for the kind-specific metadata blobs.** The schema-enforcement layer is the whole point; hand-rolled validation with `if key not in dict` is where inconsistency creeps in.
- **The successor pointer is what makes deprecation useful.** A sunset artifact without a `successor_revision_id` is a way to make the registry more confusing, not less.
- **Read-side ingest is out of scope for the exercise but not for real life.** The full read-side ingest pattern from Chapter 2 (compute hash on first read, register in background, migrate to hard cutoff later) is what makes the registry adoptable in a mature organization. Note where in your CLI the shim would live but do not build it.
- **Don't build a UI.** The CLI is enough to demonstrate the shape; a UI is a real-project affordance and not the point of the exercise.

## Acceptance criteria

- The four kind-specific validators reject at least the misconfigurations listed above with machine-readable error codes.
- The CLI's `register`, `list`, `show`, `deprecate`, and `archive` commands work end-to-end against the local SQLite/Postgres instance.
- Content hashes are stable under formatting-only diffs (whitespace, key order, trailing newline) and change under any content edit — demonstrated with a unit test.
- Registering the same content hash twice under any name is a no-op with a clear message; registering different content under the same `(name, semver)` is a failure.
- `deprecate` requires a successor pointer; `archive` requires prior `sunset` state.
- `rebaseline-check` classifies at least the fixture cases in Part D correctly and defaults ambiguous edits to major.
- `FAILURE_MODES.md` demonstrates all three failure modes with runnable commands and correct output.
- The registry emits an event stream (a JSONL log is fine) that lists every state transition, so downstream layers (the warehouse, the CI system) could subscribe.

## Stretch goals

- **Governance model.** Implement a second registration path that requires an approver actor for `major` bumps to artifacts consumed by any active release-blocking gate. The approver is looked up from a `consumers` table (which the registry could populate from Chapter 6's CI integration; the exercise stubs it).
- **Read-side ingest shim.** Build the "register on first read" shim from Chapter 2. A CLI subcommand `registry ingest --path=<file>` computes the canonical hash, checks whether a revision with that hash already exists, and if not registers it under an auto-generated name with a note that it was auto-ingested.
- **Signed artifacts.** Add a `sigstore/cosign`-style signature step to `register`: the registration produces an attestation that the content hash was signed by the registering actor's key. `show` verifies the signature on retrieval. Useful for the regulated-environment story (EU AI Act, ISO/IEC 42001).
- **Cross-artifact reference graph query.** Ship a `registry graph <artifact-ref>` command that traverses the `artifact_reference` edges and prints the transitive dependency tree — a task, the prompt it uses, the dataset it consumes, and the judge that would grade its outputs. This is the query Chapter 5's warehouse will build on.
