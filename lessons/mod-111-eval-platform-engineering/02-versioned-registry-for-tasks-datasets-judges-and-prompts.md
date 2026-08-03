# A Versioned Registry for Tasks, Datasets, Judges, and Prompts

Every reproducibility failure in an eval program eventually reduces to the same question: *which bytes did that run actually use?* An evaluation that referred to `mmlu-easy` might have used the original 2020 revision, an internal cleaned fork, or the same file with three items rewritten last Wednesday to fix a typo. A run tagged `helpfulness-judge-v2` might have used a rubric that changed twice between "v2 as of April" and "v2 as of June." A "quality gate at 0.85" derived from any of these is unfalsifiable — it is not that the number is wrong, it is that no one can say what it is a number *about*.

The eval registry is the platform layer that makes those questions answerable. It is a content-addressed store of the four kinds of artifact — tasks, datasets, judges, and prompts — with immutability, versioning, and a documented deprecation lifecycle. Every downstream layer of the platform (the orchestration plane in Chapter 3, the cost controls in Chapter 4, the warehouse in Chapter 5, the CI gates in Chapter 6) resolves its inputs *through* the registry. Without the registry, every other layer has to hand-roll its own lineage; with it, lineage is a query.

## The four registered artifact types

A minimal eval registry contains four artifact types. Each is a distinct thing that can drift independently, and each has its own versioning rules.

### Tasks

A **task** is the metadata that describes *what a specific evaluation measures* and *how it is scored*. In lm-evaluation-harness terminology, a task is a `TaskConfig` — dataset location, prompt template, target label, metric function, decoding config, few-shot policy. In Inspect terminology, a task is an `@task`-decorated function that composes a solver and a scorer over a dataset. In OpenAI evals terminology, a task is an eval YAML.

The registry stores a normalized representation of the task, independent of any single runner's file format:

- The task's stable identifier (e.g., `mmlu.stem-easy`) plus its semantic version.
- A pointer to the dataset revision(s) it consumes (see below).
- A pointer to the prompt template it uses (see below).
- The metric definition — usually a name plus a version (e.g., `exact_match@2`, `pass@1`, `judge-graded-helpfulness@1.2`).
- Decoding config (temperature, top-p, max tokens, stop sequences, seed policy).
- Runner compatibility annotations — which runners can execute this task shape, and any runner-specific translation notes.

Tasks are semantically versioned. A task version bump signals a *methodological* change: the metric definition changed, the few-shot policy changed, the decoding config changed. When the underlying dataset or prompt changes independently, that is a dataset or prompt version bump, and the task points to the new revision but the task itself may or may not bump (it depends on whether the swap is intended to preserve comparability).

### Datasets

A **dataset** is the collection of examples the task is evaluated over. In this module it includes:

- Benchmark datasets pulled from Hugging Face Hub with a specific revision SHA (the SHA is critical — a benchmark loaded from `revision="main"` will silently change under you).
- Internal datasets — production replay sets, human-authored gold sets, per-slice audit sets — versioned in the platform.
- Adversarial datasets — jailbreak prompts, red-team sets — often maintained by a safety team with a different lifecycle than the capability datasets.

The registry stores, for every dataset revision:

- The `(name, revision)` identifier plus a strong content hash of the row content (not just the file — file-level SHAs miss row-order rewrites and metadata drift).
- The row count and, if the dataset has slices, the row count per slice.
- Provenance metadata: source URL, license, curator, date of last update.
- Schema definition: which fields exist, their types, their allowed values.

A dataset revision is immutable. Row-level edits create a new revision; the old revision remains queryable so that older reports do not silently retarget.

### Judges

A **judge** is a scoring function that consumes a model output and returns a metric. LLM-as-judge is the case that dominates, but classical scorers (BLEU, ROUGE, exact match, F1) are stored in the registry too so that "scored by judge X" is a first-class field on every result.

A judge revision captures:

- The judge type: LLM-as-judge, classical, human, or hybrid.
- For LLM judges: the backend model name and version (e.g., `claude-sonnet-4-6@2026-02-01`), the rubric prompt reference (see prompts below), the decoding config, the output parser, and any post-processing (position swap, ensemble aggregation).
- For classical judges: the exact scoring function version — including which library, which version, and any parameters (BLEU-n order, ROUGE-L variant, F1 averaging).
- Calibration metadata (from mod-105): the last-known human-agreement statistic (Cohen's κ, Krippendorff's α) on a documented calibration set.

Judge versioning has a distinct failure mode that motivates a specific discipline in the registry: **judge drift**. The backend model behind a hosted-vendor judge can change without notice — an unversioned `gpt-4` route may point to a different model this month than last, and the exact same judge prompt will produce systematically different scores. The registry enforces that every judge revision pins its backend model version, and it does not accept an unversioned model route. This is the same lesson mod-110 Chapter 6 taught for observability; the registry is the platform's mechanism for making it structural.

### Prompts

A **prompt** is a versioned template — usually a Jinja-style string or a structured message list — that is composed with dataset rows and other context to produce the actual input a model is called with. Prompts appear in two places:

- **Task prompts**: the template that turns a dataset row into an input for the model under test.
- **Judge prompts**: the rubric text that turns a model output into an input for the judge.

Both live in the registry as first-class artifacts because both drift and both drift silently. A "small clarification" to a rubric that "shouldn't change anything" changes many things.

Prompt revisions store:

- The template text and its content hash.
- The templating engine and version (`jinja2==3.1.5`, `chevron==0.14.0`).
- The declared input variables and their types.
- Any embedded few-shot examples (which have their own row-level provenance).

The registry rejects a prompt revision whose declared variables do not match the placeholders in the template; that check alone catches a class of silent bugs where a rubric variable is renamed and the prompt still renders (with the variable missing) because Jinja's default is empty-string substitution.

## Content addressing: the mechanism that makes lineage possible

The single most important design property of the registry is content addressing. Every artifact revision has an identifier of the form `(name, semantic-version, content-hash)`, and downstream references resolve on the content hash, not on a mutable tag.

Two concrete implications:

- **Immutability is enforced at the storage layer, not by policy.** Once a revision's content hash is registered, its bytes cannot be replaced. A "fix" to a dataset creates a new revision with a new hash; the old revision remains queryable and every report that references the old hash continues to be interpretable. This is the same discipline Git uses for commits; the registry uses it for eval artifacts.
- **"The same eval" is a well-defined predicate.** Two runs are "on the same eval" if and only if their task hash, dataset hash, judge hash, prompt hash, and decoding config hash all match. This lets the platform answer questions like "have we ever run this exact configuration before?" with a query, and it lets Chapter 5's warehouse define the natural join key on eval results.

A content hash is not a semver replacement. Semvers communicate intent to humans (a minor bump means "you can compare across it"); content hashes communicate identity to machines. The registry uses both.

## Semantic versioning of eval artifacts

Semver as it applies to eval artifacts needs a specific interpretation because "backwards compatible" does not map cleanly onto "the same benchmark measurement." The convention:

- **Major bump**: a change that breaks comparability. New scoring metric, new few-shot policy, a substantive change to the rubric that changes what "5/5" means. Runs across a major bump *must not* be compared without a re-baseline.
- **Minor bump**: a change that preserves comparability but adds capability. Adding a new slice to a dataset where the original slices' rows are unchanged; extending a task to a new language while preserving the original English rows; refactoring a judge implementation without changing its rubric.
- **Patch bump**: a change that fixes a defect without affecting the measurement. A typo fix in the rubric that everyone (including the judge) will interpret identically; a fix to a row's `expected_output` field for an item that everyone agreed was mislabeled.

The distinction is not always easy — many changes are "we think it doesn't affect the metric but we can't be sure." The registry's convention is that ambiguous changes are treated as major bumps and require a re-baseline check: a run of both versions on a stable model to confirm the metric shift is inside noise before comparability is asserted.

## The registry's data model

A workable relational schema for the registry, expressed as tables (the underlying store can be Postgres, an object store plus a metadata index, or an existing feature-store product; the schema is what matters):

```
artifact                        -- one row per (kind, name)
  id                (uuid, pk)
  kind              (enum: task, dataset, judge, prompt)
  name              (text, unique with kind)
  owner_team        (text)
  created_at        (timestamp)

artifact_revision               -- one row per revision
  id                (uuid, pk)
  artifact_id       (fk -> artifact.id)
  semver            (text)                    -- e.g., "1.2.0"
  content_hash      (text)                    -- sha256 of canonical serialization
  storage_uri       (text)                    -- pointer to bytes (S3, git tag, ...)
  metadata          (jsonb)                   -- kind-specific structured fields
  created_at        (timestamp)
  created_by        (text)
  deprecated_at     (timestamp, nullable)
  deprecation_note  (text, nullable)
  UNIQUE (artifact_id, content_hash)

artifact_reference              -- edges of the artifact graph
  from_revision_id  (fk -> artifact_revision.id)
  to_revision_id    (fk -> artifact_revision.id)
  reference_type    (enum: uses_dataset, uses_prompt, ...)
  PRIMARY KEY (from_revision_id, to_revision_id, reference_type)
```

The `metadata` column carries the kind-specific structure — a judge revision's row carries the backend-model version and rubric-prompt pointer; a task revision's row carries the decoding config and metric definition. Chapter 5's warehouse joins on the `artifact_revision.content_hash` column, which is why it is the natural key.

## Deprecation as a first-class lifecycle

A registry that never deprecates an artifact grows without bound; every capability team adds five new judge variants a quarter and the "which judge should I use" question devolves back into tribal knowledge. Deprecation is what keeps the registry usable.

The lifecycle:

- **Active**: the revision is a recommended choice for new work. It appears in default listings and in the UI's "current versions" view.
- **Sunset**: the revision is superseded but still usable. New work is discouraged (the UI shows a "prefer X instead" pointer); existing work continues to reference it and its results remain reproducible.
- **Archived**: the revision is no longer executable by the platform (the runner adapters do not target it). Historical results remain queryable in the warehouse; new runs against archived revisions fail with a clear error.

Deprecation is not deletion. Once a revision has been used to produce a published metric, its bytes are retained for the platform's audit horizon (Chapter 5 covers retention). Removing it would invalidate the lineage of every downstream report.

Two operational rules keep the lifecycle honest:

- **Deprecation with a successor pointer**. Every sunset artifact points at its recommended successor. `helpfulness-judge@1.2` sunsets with `see helpfulness-judge@1.3 (adds no-op rubric clarifications, MDD-verified compatible)`. The pointer is what turns "which judge do we use" from tribal knowledge into a query.
- **Deprecation on a schedule**. Sunset lasts at least until every consuming CI gate has migrated. The registry surfaces "who consumes this revision" in a query so the sunsetting team knows who to notify; Chapter 6 wires that surface into the release pipeline's dependency check.

## Governance: who can register, who can deprecate, who reviews

Three governance patterns work in practice; pick one and commit.

- **Central curation**: a single platform team owns all four artifact types and reviews every registration. This is the highest-trust and lowest-velocity model. Appropriate for organizations under strong regulatory pressure or for artifact types where drift is very high-cost (safety judges, contamination canaries).
- **Federated curation with review**: each capability team owns the artifacts in its domain (the safety team owns safety judges, the RAG team owns retrieval datasets) but changes go through a lightweight review. Appropriate for medium-sized organizations where teams have deep domain expertise the platform team lacks.
- **Wide-open registration with reputation**: any authenticated user can register; the platform surfaces usage counts and last-modified dates so that consumers can vote with their references. Appropriate only for research-heavy organizations where speed of iteration is the dominant constraint.

Whichever model is chosen, the registration API enforces the schema (a judge without a backend-model version is rejected; a dataset without a license field is rejected; a prompt whose template variables do not resolve is rejected). Schema enforcement is not review — it is the minimum bar below which the artifact cannot exist.

## Migration: importing an existing zoo of ad-hoc artifacts

Most organizations arrive at the registry mid-flight, with dozens of judges in a Slack thread, hundreds of datasets in a git monorepo, and thousands of prompt templates spread across five services. A workable migration pattern:

- **Identify the highest-leverage artifact type first.** Judges are usually the answer — they have the highest drift cost and the smallest inventory. Datasets are second. Prompts are broadest and can be migrated in tranches once the discipline is established.
- **Introduce read-side compatibility.** The first migration step is not "everyone must register their artifacts"; it is "the registry can *ingest* existing artifacts on first read." When a task is submitted that references `helpfulness_v2.txt`, the registry computes its content hash, registers it under a canonicalized name, and returns a stable identifier. Existing consumers keep working; the registry acquires ground truth in the background.
- **Roll a hard cutoff by artifact type.** Once the read-side ingest has been active long enough that most artifacts are already present in the registry, flip a switch: submissions that do not resolve through the registry are rejected. This is the discipline moment; it usually needs an announcement, a migration doc, and a support channel for a week or two.

The temptation to build the perfect registry-shaped world first and migrate everything into it at once is very strong and it will fail; every attempted big-bang eval-registry migration this author has seen has been abandoned or descoped. Incremental is the only strategy that ships.

## Failure modes the registry is designed to prevent

Three specific failure modes recur across organizations that don't have a real registry, and the registry designs above are targeted at them.

### Failure mode: "same benchmark" is silently a different benchmark

Team A runs `mmlu` from Hugging Face's default revision; team B runs `mmlu` from an internal fork with three items rewritten; team C runs `mmlu` via an adapter that skips the "Elementary Mathematics" subject "because it was noisy." All three publish an "MMLU score." None are comparable.

Registry mitigation: every dataset reference resolves through a content hash. If team B's fork has a different hash than team A's, they are different datasets and the registry names them differently. The three scores are annotated with their dataset hash in the warehouse; a query for "compare our MMLU scores" surfaces the hash mismatch immediately.

### Failure mode: judge drift makes yesterday's score incomparable with today's

A judge configured against a hosted-vendor route ("gpt-4") produces one score in March and a different score in June because the vendor rolled the underlying model. The score has not been recomputed; the incumbent's baseline is now measured against a different judge than the candidate is. Every quality gate downstream is unfalsifiable.

Registry mitigation: every judge revision pins a backend-model version, and the registry rejects unversioned model routes at registration time. When the vendor releases a new model, the registry receives a new judge revision — the old one continues to work as long as the vendor keeps the old model routable (a separate SLA concern) and clearly stops working when they don't.

### Failure mode: a "small clarification" to a rubric silently moves every score

A judge rubric is edited to "clarify" a scoring anchor. The edit is intended to preserve the metric's meaning. It doesn't — one anchor's meaning shifts by half a point, and every subsequent score is systematically higher. The clarification is not versioned because it "isn't a real change."

Registry mitigation: every prompt revision has a content hash. An edit — any edit — produces a new revision, and the platform routes downstream consumers through a re-baseline check. The rubric edit either bumps as a patch (with a supported re-baseline note showing no measurable shift) or bumps as major (with an explicit signal to consumers that comparability across the boundary is not asserted).

## Guidance for the platform engineer

- **Start with the four artifact types, not with the runner.** A runner abstraction over unversioned artifacts leaks lineage debt at every layer above; a registry with a single-runner backend accumulates value from day one.
- **Content hashes on every artifact.** File-level SHAs are not enough; canonicalize the serialization first (JSON with sorted keys, LF line endings, no trailing whitespace) so that formatting changes do not appear as content changes.
- **Reject unversioned vendor model routes.** The single-line judge configuration `model: "gpt-4"` (unversioned) is the largest reproducibility bug this module can prevent. Reject at registration; refuse to run.
- **Deprecation with a successor pointer.** A sunset artifact without a pointer to what to use instead is a way to make the registry harder to use, not easier. Point downstream users at the right place.
- **Read-side ingest before hard cutoff.** Every big-bang migration this author has seen has failed; every incremental migration has shipped.
- **Owner metadata is queryable.** For any artifact, "who owns this and who consumes it" must be a query, not a Slack expedition. Chapter 5's warehouse joins on this metadata.

## Summary

The versioned registry is the load-bearing foundation of the eval platform: without it, no downstream layer can offer real lineage, real comparability, or real reproducibility. It stores four artifact types — tasks, datasets, judges, and prompts — each with content-addressed immutable revisions and a documented deprecation lifecycle. Semver communicates intent to humans; the content hash communicates identity to machines; downstream references resolve on the hash. Governance can be central, federated, or reputational, but schema enforcement (backend-model pinning for judges, license and provenance for datasets, resolvable template variables for prompts) is a floor, not a policy. Migration to the registry is incremental and starts with the highest-drift artifact type — usually judges. The three failure modes the registry structurally prevents are silent dataset divergence, judge drift under hosted vendors, and unversioned rubric edits. With the registry in place, the next chapter unifies the concrete runners — lm-evaluation-harness, Inspect, OpenAI evals — behind a single orchestration plane that resolves its inputs through the registry.
