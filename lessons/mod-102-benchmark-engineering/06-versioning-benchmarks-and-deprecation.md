# Versioning Benchmarks and Deprecation Policy

Every prior chapter produces an artifact that could change: raw data revisions (Chapter 1), schema changes (Chapter 2), instruction revisions (Chapter 3), gold rotations (Chapter 4), and contamination-driven item rotations (Chapter 5). Without a versioning policy, each change silently moves scores. Chart-toppers become chart-losers overnight and no one can tell whether the model got worse, the scorer got fixed, or the eval got harder. This chapter is the discipline that keeps benchmark scores comparable over time — a hashed, semantically-versioned manifest and a deprecation policy that says what to do when the benchmark can no longer measure what it was built to measure.

## Three hashes, one manifest

A benchmark's identity for reproducibility purposes is captured by three hashes that together determine any score anyone will report on it. Each hash is computed deterministically from a specific set of files; jointly they live in `MANIFEST.json`.

- **`dataset_hash`.** SHA-256 over the sorted concatenation of the per-item content hashes plus the `splits.yaml` content. Changes when items are added, removed, edited, or reassigned to a different split.
- **`task_hash`.** SHA-256 over `task.yaml`, canonicalized (YAML → JSON canonical form) so whitespace and key ordering do not affect the hash. Changes when the prompt template, allowed outputs, scoring reference, aggregation rule, or decoding config change.
- **`evaluator_hash`.** SHA-256 over the `evaluator.py` source (or the pinned wheel), plus the pinned versions of any subgraders it calls. Changes when the scoring code changes. For LLM-as-judge evaluators, includes the judge model version, judge prompt, and judge decoding config as separate pinned fields.

The manifest is a small JSON file that ships with the benchmark:

```json
{
  "name": "internal-support-triage",
  "version": "1.2.0",
  "dataset_hash": "sha256:8f3c…",
  "task_hash": "sha256:1a92…",
  "evaluator_hash": "sha256:4bd0…",
  "counts": {
    "dev": 500,
    "public_test": 2000,
    "private_test": 2000,
    "canary": 100
  },
  "contamination_report": "reports/contamination-v1.2.0.json",
  "contamination_report_hash": "sha256:c07e…",
  "released_at": "2026-08-01T00:00:00Z",
  "previous_version": "1.1.0"
}
```

The manifest is versioned in the benchmark repo and copied into every score record downstream code produces. A score without a manifest reference is not comparable to anything; a score with one can be joined against past runs to reason about change.

## Semantic versioning for benchmarks

Semantic versioning (semver.org) for software applies almost directly to benchmarks, with three adjustments for what "breaking" means in the eval context.

- **`MAJOR`** — the construct or the primary aggregation changed. Scores from the new version are *not* comparable to old-version scores. Example: expanding the label set from binary to 6-class; changing the primary metric from accuracy to weighted-F1; splitting the benchmark into per-domain sub-benchmarks. Bump the major; retire the old headline number.
- **`MINOR`** — the dataset, task spec, or evaluator changed in a way that *can* shift scores but the construct and headline metric are the same. Item rotations, prompt-template polishing, scoring bug fixes, evaluator refactors. Scores are cross-comparable *only* with a documented delta (see below).
- **`PATCH`** — non-scoring changes. Typo fixes in the README, additional documentation, reformatting of the manifest, reproducibility improvements. Score-neutral by construction. If a patch would move any score, it is a MINOR, not a PATCH.

Two operational rules that keep the versioning honest.

**A change to any of the three hashes is at least a MINOR bump.** If the hash moved and the version did not, the release is malformed. CI should assert this.

**Every MINOR release ships a delta report.** For a fixed reference model (or panel of models), the delta report shows the per-slice score change between vN and vN+1 on the intersection of items. Readers use it to correct historical comparisons. Without it, "we rotated 3% of items" is unfalsifiable and unauditable.

## What triggers each kind of bump

Concrete triggers, mapped to chapters that produce them:

- **Chapter 1: License / provenance correction on an item.** Usually a PATCH (metadata-only), unless the item is dropped for consent reasons — then MINOR because `dataset_hash` moves.
- **Chapter 2: Schema addition of a nullable metadata field.** PATCH. Schema removal or a change to `input`/`reference`/`item_id` semantics: MAJOR.
- **Chapter 3: Instruction revision that clarifies but does not change labels.** PATCH if no gold labels moved; MINOR if re-adjudication moved labels (`dataset_hash` moves).
- **Chapter 4: Gold rotation that revises labels on a sample.** MINOR.
- **Chapter 5: Item rotation to replace contaminated items.** MINOR.
- **Task-spec prompt template change.** MINOR at a minimum; MAJOR if the template change is a construct expansion (e.g. moving from zero-shot to few-shot as the canonical mode).
- **Evaluator bug fix.** MINOR. The delta report must show the pre/post score change per slice for the reference model.
- **Adding a new slice column to `aggregation.slices`.** PATCH (existing headline unchanged; new slice is additional).
- **Changing the headline metric.** MAJOR.

For borderline cases, err MAJOR. Under-versioning silently breaks comparability; over-versioning is loud and cheap to fix in the changelog.

## The changelog format

Every release ships a changelog entry, not just a semver number. The entry names what changed, why, the affected counts, and the reference-model delta. Keep it structured:

```yaml
version: 1.2.0
released_at: 2026-08-01T00:00:00Z
previous_version: 1.1.0
change_type: MINOR
changes:
  - kind: item_rotation
    reason: contamination_flagged_v1.1_report
    items_removed: 47
    items_added: 47
    slice_impact:
      billing: -3.2 pp accuracy on reference model gpt-x-baseline
      technical: -0.1 pp accuracy on reference model gpt-x-baseline
  - kind: instructions_revision
    reason: adjudicator_log_edge_cases
    instructions_version: 1.1.0 -> 1.2.0
    items_relabelled: 12
delta_report: reports/delta-v1.1.0-to-v1.2.0.md
migration_guidance: >
  Re-score models on v1.2.0. Do not compare v1.2.0 headline to v1.1.0
  headline; compare on the intersection using the delta report.
```

The delta report is the operational bridge; the changelog is the human-facing explanation.

## Deprecation policy

Every benchmark has an end. Deprecation is the policy for how it ends without leaving downstream consumers stranded.

Triggers that should start a deprecation clock:

- **Score saturation.** Frontier models score above the inter-annotator agreement ceiling for two consecutive releases. The benchmark can no longer separate frontier from frontier; it is measuring noise.
- **Pervasive contamination.** Chapter 5's report shows > `X`% of items flagged and rotation would replace a large majority. Beyond a threshold, rotation is more expensive than starting a successor.
- **Construct drift.** The construct itself has moved (product change, domain shift, new harm categories) and adapting the current benchmark would be a MAJOR bump that produces something too different from the current benchmark to be usefully called the same thing.
- **Legal / licensing changes.** A source's terms change; a data-subject withdraws consent for training-eval use.
- **Better successor exists.** A newer benchmark measures the same construct with fewer known threats.

The deprecation lifecycle:

1. **Announce deprecation.** In the release notes and in the manifest (`deprecation_status: "deprecated", deprecation_reason: "…", successor: "successor-name-vX.Y.Z"`). Give the deprecation date and the sunset date.
2. **Freeze the benchmark at the deprecation version.** No further MINOR or MAJOR releases; only PATCH-level documentation may change. This gives downstream users a stable target to hit while they migrate.
3. **Maintain for a stated support window.** Typically two release cycles or six months, whichever is longer. During the window, contamination and delta reports may continue as advisory but the item set and task do not change.
4. **Sunset.** The benchmark artifact is archived (kept queryable for reproducibility of past scores) but no longer recommended for new work. The manifest carries `deprecation_status: "sunset"` and the successor's identifier.

The support window is the point of the policy. A benchmark that disappears the day it is deprecated forces its consumers into an unplanned migration; a benchmark that lingers forever tempts them into indefinite comparison against a moving landscape of models on a frozen construct. The declared window makes the trade-off explicit.

## Rotation policy: what to keep, what to swap

Rotation is the ongoing MINOR-level defense against contamination and gold drift. A policy worth writing down:

- **Cadence.** Every N months, or on every contamination report that raises flagged items above a threshold.
- **Rotation rate.** Typically 5–15% of items per rotation. Too low and contamination stays ahead; too high and the delta report becomes so large that historical comparison is impractical.
- **Selection rule.** Prioritize (a) items flagged by any contamination detector at any severity; (b) items with the most model score variance across recent runs (usually the most-adapted-to items); (c) items whose gold labels changed in the last rotation cycle. Stratify replacements to preserve the slice distribution.
- **Backfill sourcing.** Draw replacements from data collected after the reference model cutoff, from sources whose license permits eval use, with the full Chapter 1 provenance record.
- **Canary integration.** Every rotation adds a small number of canary items (Chapter 7) so future contamination probes have material to work with.
- **Delta reporting.** Publish the pre/post score change for a reference model panel. Consumers can decide whether the shift is acceptable for their comparison.

## Publishing versions: leaderboards, papers, model cards

The versioning discipline is only as good as the consumers who use it. Push the manifest into the reporting surface:

- **Leaderboards.** Require the version string in every submission. Show version alongside the score. Do not stack rank across versions; if you must show multiple versions on the same leaderboard, use a version-explicit column.
- **Papers.** Cite the benchmark by name *and* version *and* the three hashes (or a stable identifier that resolves to them, such as a Zenodo DOI). "MMLU" without a version is ambiguous once MMLU has been re-issued or corrected.
- **Model cards.** For every eval reported, name the version, the manifest URL, and the date the eval was run. Include the contamination-report URL for the specific version-model pair.
- **Internal dashboards.** Store the manifest hashes alongside every logged score. A regression detector that compares scores across weeks needs to filter by manifest version, not just benchmark name; otherwise a benchmark bump looks like a model regression.

## What CI enforces

Automate what the policy states. A minimal CI:

- **Hash-consistency check.** On every PR to the benchmark repo, re-derive `dataset_hash`, `task_hash`, `evaluator_hash` and assert they match the values in `MANIFEST.json`. Divergence means either a stale manifest or an unsound change.
- **Version-bump check.** If any of the three hashes changed, assert the semver `version` bumped by at least the expected level (MINOR or MAJOR).
- **Changelog presence.** Any version bump requires a corresponding structured changelog entry.
- **Delta-report presence.** MINOR/MAJOR bumps require a delta report file at the path named in the changelog.
- **Deprecation banner.** If the manifest carries `deprecation_status`, README and leaderboard rendering must display the banner.

Manual review is where the change-type call (PATCH vs. MINOR vs. MAJOR) is exercised. CI catches the mechanical mistakes; the reviewer catches the semantic ones.

## A worked micro-example

Suppose you ship `v1.1.0` of a support-triage benchmark. A month later:

- Chapter 5's contamination report flags 40 items as likely-contaminated for the frontier model you're evaluating.
- The Chapter 4 gold-rotation cycle finds 8 items where the label needs revision under the new instructions.

You cut a `v1.2.0`:

- Rotate the 40 contaminated items with 40 freshly-collected items from the past three months of production traffic (Chapter 1 provenance recorded, consent flag intact).
- Re-adjudicate the 8 items, updating gold.
- Update `dataset_hash` (item set changed) and record `previous_version: 1.1.0`.
- `task_hash` and `evaluator_hash` unchanged.
- Compute delta on the intersection (2000 items in v1.1.0 minus the 40 removed minus the 8 relabelled = 1952 stable items) for a reference model panel. Publish `reports/delta-v1.1.0-to-v1.2.0.md`.
- Push the changelog entry.

Consumers know exactly what changed, can re-score their models, and can compare against `v1.1.0` numbers using the delta report or the intersection. If in six months you find contamination has advanced faster than rotation, you cut `v2.0.0` — a new construct-scoped benchmark — and deprecate `v1.x` per the policy above.

## What versioning does not fix

- **Backwards-comparability of scores across MAJOR versions.** A construct change is a construct change; no versioning trick makes v1 and v2 scores directly comparable. That is the point of the MAJOR bump.
- **Silent drift in external dependencies.** A pinned LLM-judge model that the vendor swaps under the hood (mod-105 chapter on judge drift). The evaluator hash names the pinned model; you still have to verify the vendor honored the pin, or repoint to a self-hosted equivalent.
- **Score misinterpretation.** A well-versioned benchmark reported without its uncertainty (mod-101 Chapter 3–4) is still misleading. Versioning is necessary, not sufficient.

## Summary

A benchmark's reproducible identity is three hashes — dataset, task, evaluator — bundled into a manifest and semantically versioned. MAJOR bumps mark construct changes and break backwards comparability by design; MINOR bumps mark score-shifting changes to items, task spec, or evaluator and always ship a delta report against a reference model panel; PATCH bumps are score-neutral. Enforce the hash-manifest-changelog relationship in CI so under-versioning cannot ship silently. Rotate items on a cadence that stays ahead of contamination without making historical comparison useless. When a benchmark saturates, drifts, or leaks past a threshold, deprecate cleanly with a stated support window and a named successor. Chapter 7 turns to the split-level defenses — public and private holdouts, and canary items — that give the versioning discipline something to protect.
