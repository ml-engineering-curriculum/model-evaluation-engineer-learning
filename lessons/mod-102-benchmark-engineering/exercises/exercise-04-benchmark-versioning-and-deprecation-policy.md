# exercise-04: Benchmark Versioning and Deprecation Policy

**Estimated effort:** 3 hours

## Objective

Implement the three-hash manifest for a small benchmark you own, walk through a `v1.0.0 → v1.1.0` MINOR rotation with a delta report, walk through a `v1.1.0 → v2.0.0` MAJOR cut, and write the deprecation policy that governs both. The output is a working versioning tool plus a signed-off policy document — the pair a benchmark owner would keep in the repo for a lawyer, an auditor, or a downstream consumer to read.

The point is to feel the constraint semantic versioning puts on eval work: which changes force a MINOR (and therefore a delta report), which force a MAJOR (and therefore give up backwards comparability), and which are pure PATCH (and therefore must not move any score).

## Prerequisites

- Chapter 06 of this module.
- Chapter 02 (schema) and Chapter 05 (contamination) are useful — this exercise assumes you have a small benchmark artifact in front of you.
- Python 3.11+ with `pyyaml` and `hashlib` (standard library is enough).

## The starter benchmark

Use one of:

- **The gold set from exercise-02.** Ideal — you already own the items, the task spec, and the evaluator.
- **A synthetic 50-item benchmark you construct in 15 minutes.** For instance, a text-classification eval with two classes, a fixed prompt template, and an exact-match evaluator. Real data is better; synthetic is acceptable if it lets you focus on the versioning mechanics.
- **A publicly-hosted small benchmark** you clone into your working directory and copy the necessary files from (with license attribution).

The benchmark artifact must contain, at minimum:

```
benchmark/
  MANIFEST.json
  task.yaml
  evaluator.py
  splits.yaml
  data/
    dev.jsonl
    public_test.jsonl
    private_test.jsonl        # can be a shim; used to show the manifest hash logic
    canary.jsonl              # can be a shim
  ATTRIBUTION.md
  README.md
  CHANGELOG.md
```

## Requirements

### Part A — implement the manifest computer

Write `manifest.py` with the following functions:

```python
def dataset_hash(data_dir: pathlib.Path, splits_yaml: pathlib.Path) -> str:
    """SHA-256 over the sorted concatenation of per-item content hashes
    for every split file, plus the splits.yaml content."""

def task_hash(task_yaml: pathlib.Path) -> str:
    """SHA-256 over task.yaml in canonical form (YAML→JSON, sorted keys)."""

def evaluator_hash(evaluator_py: pathlib.Path, pinned_deps: dict[str, str]) -> str:
    """SHA-256 over the evaluator source plus a serialization of pinned
    dep versions (name, version, wheel hash if available)."""

def compute_manifest(benchmark_dir: pathlib.Path) -> dict:
    """Compute and return the full manifest dict for the benchmark at
    benchmark_dir. Includes name, version, the three hashes, per-split
    counts, released_at, and previous_version if present."""

def write_manifest(benchmark_dir: pathlib.Path) -> None:
    """Compute the manifest and write it to benchmark/MANIFEST.json.
    Fail if the version in the existing MANIFEST.json doesn't match a
    semver bump that matches the hash changes (see Part D)."""
```

- Canonicalize YAML by loading and dumping via `json.dumps(..., sort_keys=True, ensure_ascii=False, separators=(",", ":"))` so YAML formatting doesn't affect the task hash.
- Per-item content hash is `SHA-256(canonical_json_of_row)` where `canonical_json_of_row` uses sorted keys and no trailing whitespace. Include the `item_id` in the hash input.
- `pinned_deps` for the evaluator should include at least the Python version and the `scikit-learn` / `numpy` / whatever versions the evaluator imports.

### Part B — run the v1.0.0 release

- Cut `v1.0.0` of your starter benchmark.
- Run `write_manifest`; commit `MANIFEST.json`, `CHANGELOG.md` entry, and a git tag `benchmark/v1.0.0`.
- Publish the manifest hashes in `README.md`.

### Part C — cut v1.1.0 (MINOR rotation)

Make the following changes, each of which should force at least a MINOR bump under Chapter 6's rules:

1. **Item rotation.** Remove 5 items and add 5 new items (with full Chapter 1 provenance). Update `data/public_test.jsonl` and `splits.yaml` counts.
2. **Instruction revision.** Bump `INSTRUCTIONS_v1.md` to `v1.1.0` and re-adjudicate 3 items whose gold label changes under the new rules.
3. **Evaluator bug fix.** Add a small bug fix to `evaluator.py` (e.g. add whitespace stripping, or fix a case-sensitivity bug). Bump the evaluator's internal version.

Re-run `write_manifest`. The manifest version should now be `1.1.0`; the three hashes should all have changed.

Produce `reports/delta-v1.0.0-to-v1.1.0.md` containing:

- Item count change (removed / added / unchanged / relabelled).
- Reference-model delta. Pick one open-weight model, or an API you have access to, and run the benchmark under both v1.0.0 and v1.1.0. Report per-slice and overall score changes on the intersection of items. If you cannot run a model, simulate a reference-model panel with a small rule-based baseline (e.g. always-majority-class) — the mechanics are what matters.
- Rationale for the change type (why MINOR, not MAJOR).
- Migration guidance for downstream consumers.

Push the changelog entry per Chapter 6's format.

### Part D — CI enforcement

Add a `ci_check.py` script (or a GitHub Actions workflow YAML) that runs on every PR to the benchmark repo. It must:

1. Recompute the three hashes from the source files and assert they match `MANIFEST.json`. Fail loudly if not.
2. If any hash changed relative to the previous commit, assert the `version` in the manifest bumped by at least MINOR (i.e. patch component alone cannot change if a hash changed).
3. If the version bumped, assert `CHANGELOG.md` has a corresponding new entry with the expected version.
4. If the change is MINOR or MAJOR, assert the delta report file named in the changelog exists.

Include a `tests/` directory with at least two tests that exercise the CI script's pass and fail paths.

### Part E — cut v2.0.0 (MAJOR construct change)

Make a change to the benchmark that is a genuine MAJOR bump:

- Expand the label set from binary to multi-class *or* change the primary aggregation (e.g. from accuracy to macro-F1) *or* replace the prompt template with a substantively different one.

Cut `v2.0.0`. The `previous_version` field points to `v1.1.0`. The changelog entry names the MAJOR reason, and there is *no delta report* (or a very short one that explicitly explains why cross-version comparison is not meaningful — that is the point of MAJOR).

### Part F — write the deprecation policy

Produce `DEPRECATION_POLICY.md` (target: 600–1200 words) that answers:

1. **Trigger conditions.** Under what circumstances would this benchmark be deprecated? (Score saturation, pervasive contamination, construct drift, licensing changes, better successor.) Give concrete thresholds where you can.
2. **Deprecation lifecycle.** Announce → freeze → support window → sunset. State the support window and the criteria for extending or shortening it.
3. **Successor policy.** How a successor is announced, when it is required (MAJOR bump of an existing benchmark is usually the successor path), and how the manifest names it.
4. **Communication.** How you tell downstream consumers, what banners appear in the README and on the leaderboard, and what artifacts remain accessible after sunset.
5. **Reproducibility guarantee.** After sunset, what parts of the benchmark remain queryable so that past scores can still be reproduced.
6. **Exceptions and appeals.** How a consumer with a specific dependency can request an extension or a re-release. Who decides.

The policy is written for a hypothetical organization that owns and ships several benchmarks. It should be one document that applies to all of them; benchmark-specific details go into per-benchmark READMEs.

## Starter guidance

- The canonical-JSON step matters more than it looks. Two YAML files that render identically to a human can hash differently because of key ordering, whitespace, or Unicode normalization. Test your canonicalization by round-tripping and asserting the byte-identical output.
- Do not include timestamps or environment-specific fields in what you hash. `released_at` is a manifest field, not a hash input.
- The per-item content hash needs to include the `item_id`, or two items with the same content but different IDs will collide.
- Semantic-version choices for Part C are the pedagogic point. Talk through each of the three changes in the writeup: for each, argue whether it *alone* would be a MINOR (item rotation and instruction rewrite are unambiguously MINOR; an evaluator bug fix that changes any score is also MINOR).
- For the reference-model delta in Part C, do not skip it. Even a rule-based baseline demonstrates the mechanics; the report is what makes rotations auditable.
- The CI checks in Part D are a small amount of code but they are what keeps the versioning discipline honest. In a real repo, treat this workflow as blocking.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- `manifest.py` runs on the starter benchmark and produces a `MANIFEST.json` whose three hashes are reproducible byte-identically from a clean checkout.
- Both v1.1.0 (MINOR) and v2.0.0 (MAJOR) are cut cleanly with correct manifests, changelog entries, and (for v1.1.0) a delta report.
- The CI script fails on a synthetic PR that (a) changes a data file without bumping the version, (b) bumps the version without a changelog entry, and (c) makes a MINOR bump without a delta report.
- The delta report includes at least one reference-model number (real or baseline) with a per-slice breakdown.
- The MAJOR bump explicitly does *not* claim cross-version comparability.
- `DEPRECATION_POLICY.md` covers all six requirement sections and states concrete triggers and a specific support window.
- All version bumps have a git tag (`benchmark/vX.Y.Z`) and the tag matches the manifest version.

## Stretch goals

- **Sign the manifest.** Extend `manifest.py` to produce a detached signature (via `sigstore`, `minisign`, or GPG) and verify it in CI. This is what a production benchmark that ships to regulated environments would do.
- **Cross-repo manifest resolution.** Publish the benchmark manifest to a small static site (GitHub Pages) or an OCI registry and write a resolver that, given a benchmark name + version, fetches the manifest and verifies the hashes against the artifact URL. Model on how container images are resolved by tag + digest.
- **Multi-benchmark policy.** If you own multiple benchmarks in the exercise, factor common policy into a `POLICY.md` and per-benchmark overrides. Show how the CI script consumes the shared policy.
- **Rotation simulation.** Simulate 4 quarterly rotations (v1.1, v1.2, v1.3, v1.4) with 5% item rotation each; compute cumulative drift against v1.0 and discuss when the cumulative delta would motivate a MAJOR.
- **Automated deprecation banner.** Wire the manifest's `deprecation_status` field into the README rendering so a deprecated benchmark shows a prominent banner without a human editing the file.
