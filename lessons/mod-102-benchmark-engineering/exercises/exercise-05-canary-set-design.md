# exercise-05: Canary Set Design

**Estimated effort:** 2 hours

## Objective

Design a canary set for a benchmark you own — both a public canary batch and a private canary batch — build the registry that catalogs them, and specify the periodic probe that detects post-hoc training-set inclusion. Ship the design as a small artifact (a canary JSONL, a registry file, and a probe runbook) that a future you (or a colleague) could operate for the next several years.

The point is to make the "how would we know if this benchmark ended up in a training corpus" question answerable by construction, not by post-hoc guessing.

## Prerequisites

- Chapter 07 of this module.
- Chapter 05 (contamination detection) — the probe mechanics reuse the behavioral detectors from that chapter.
- Python 3.11+ with `secrets` (standard library, for generating high-entropy tokens) and access to an LLM you can query for text completion or log-likelihood.

## The target benchmark

Use one of:

- **The benchmark you built in exercise-02 or exercise-04.** Best — you already understand its construct and can add canary items that fit.
- **A synthetic 20-item benchmark you construct for this exercise.** Acceptable if it lets you focus on the canary design.

You need write access to `data/canary.jsonl` and to a private artifact store (a local `private/` directory is fine for the exercise).

## Requirements

### Part A — design the public canary batch

Produce `data/canary_public.jsonl` with 20 items. Each public canary is a benchmark item whose distinguishing token is a high-entropy string that:

- **Is unique.** Generated with `secrets.token_urlsafe(24)` or equivalent; long enough that a random collision in web text is effectively zero.
- **Is greppable.** Appears verbatim in the item's `input` and/or `reference`, in an unambiguous surrounding context ("The special code CANARY-XXXXX-YYYYY refers to …").
- **Is filterable.** The canary token pattern is publicly documented (e.g. "any string matching regex `CANARY-[A-Z0-9]{5}-[A-Z0-9]{5}` is a canary and should be excluded from training corpora"). Downstream training operators can honor the filter.
- **Is task-relevant.** The item still exercises the benchmark's construct — a canary that is transparently a canary and nothing else is a wasted slot.

Include a `canary_type: public` field in the item metadata and a `canary_token` field with the token itself.

Write `CANARY_PUBLIC_NOTICE.md` (150–300 words) that would ship in the benchmark README:

- The canary token regex.
- An explicit request that training pipelines filter these items.
- The rationale (why the request is being made) and a citation to a precedent for this practice (e.g. BIG-bench).

### Part B — design the private canary batch

Produce `private/canary_private.jsonl` with 20 items. Each private canary is a benchmark item whose distinguishing token is high-entropy and unique, but:

- **The token is not publicly disclosed.** The item appears in `public_test` looking like any other item. There is no public notice that this item is a canary. Downstream training operators cannot filter for it.
- **The registry entry lives only in the private artifact store.** Reveal it only during a probe (Part D) or an audit.
- **The token must survive paraphrase or accidental rewriting.** Embed it structurally (e.g. as a named entity that is central to the item's answer, not as a throwaway parenthetical), so that a corpus that ingested the item is unlikely to have dropped the token.

Include `canary_type: private` (in the *private* registry only — not in the shipped item's metadata; the shipped item looks generic).

### Part C — the canary registry

Produce `private/canary_registry.yaml` with one entry per canary (public and private both):

```yaml
- canary_id: canary-001
  canary_type: public
  canary_token: CANARY-A7X92-QK4M1
  item_id: item-8f2a
  benchmark_name: internal-support-triage
  benchmark_version_at_introduction: 1.2.0
  created_at: 2026-08-03T00:00:00Z
  distinguishing_context: >
    Token appears in the ticket body as a purported error code that
    the ticket asks the model to explain.
  probe_last_run: null
  probe_history: []
- canary_id: canary-021
  canary_type: private
  canary_token: ZKR8T-4H29Q-P0EJ7
  item_id: item-4c11
  ...
```

The registry lives in the private artifact store. Access to it is logged.

### Part D — the probe runbook and script

Write `probe.py` and `PROBE_RUNBOOK.md`.

`probe.py` implements at least two of the probes below and runs against a specified model:

- **Verbatim completion probe.** For each canary, prompt the model with the first `k` tokens of the canary item (up through, but not including, the canary token) and check whether the model completes it to include the canary token verbatim. If yes, log a hit.
- **Log-likelihood probe.** For each canary item, compute the model's log-likelihood on the item. Compare against a matched control: an item of similar length and structure drawn from data collected *after* the model's training cutoff, that never appeared in your benchmark. A statistically-significant likelihood gap (paired-bootstrap CI on the mean-per-token log-likelihood difference, mod-101 Chapter 4) is a memorization signature.
- **Distinguishing-token likelihood probe.** For each canary item, compute the model's probability of generating the canary token *given the surrounding context*. Compare against the probability of the same token conditioned on unrelated prefixes. A conditioning-specific spike is a memorization signature.

`PROBE_RUNBOOK.md` (400–800 words) describes:

- Cadence: how often to run the probes (quarterly is a reasonable default).
- Which models to probe (every model your organization evaluates on this benchmark, plus any frontier release).
- Escalation: what to do when a probe returns a hit (publish; rotate; re-annotate; feed back into Chapter 5's contamination report).
- Rotation rule: retire a specific canary after a confirmed hit, add replacements from the next benchmark MINOR bump.
- Reproducibility: the seed, the control corpus, and the exact prompt template used by the probe.

### Part E — run the probe once

Run `probe.py` against at least one model whose training cutoff you know. Log the results.

- Expected outcome: probes return no hits (your canaries are fresh; the model was trained before they existed). If a probe does return a hit, that is interesting data — inspect it (false positive? actual leakage?) and note in the writeup.
- Report the probe's discriminative power on a *positive control*: create a "known contamination" scenario by fine-tuning a tiny model on a mock canary item (or, without fine-tuning access, by including a mock canary in the prompt context as an in-context example) and verify the probe fires.

### Part F — the writeup

Produce `CANARY_DESIGN.md` (800–1500 words) that:

1. Names the target benchmark and version at canary introduction.
2. Describes the public and private canary designs, with justification for the token entropy and the structural placement.
3. States the false-positive and false-negative expectations for each probe you implemented, based on the positive-control result.
4. Lays out the rotation plan for the next four release cycles.
5. Discusses what the canary set *cannot* tell you (paraphrased ingestion, distribution-level contamination rate, non-verbatim reasoning-chain memorization).

## Starter guidance

- Use `secrets.token_urlsafe(24)` for token generation, not `random.random()` — the security-grade RNG is what makes collision-freedom defensible.
- Do not reuse a canary token across benchmarks; the whole point is that the token is a unique identifier.
- For a private canary, resist the temptation to make the token an obvious placeholder ("XYZ-CONTAMINATION-CHECK-42"). The private canary must look like ordinary content to a crawler.
- The public canary notice is a courtesy that only works if honored, and it is only honored if it is discoverable. Put it in the benchmark's main README, not buried in an appendix.
- The private artifact store for the exercise can be a local directory ignored by git. In a real setting, it would be an access-controlled bucket with logged access. Note this in the writeup.
- The positive control in Part E is what proves the probe has power. Without it, "we ran the probe and got no hits" is unfalsifiable.
- The matched control for the log-likelihood probe is the hardest part. Items you generate yourself post-cutoff, or items from a dataset you know was released after the model, are the two workable options. Cite the source and date.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- 20 public canaries in `data/canary_public.jsonl` and 20 private canaries in `private/canary_private.jsonl`, each with a unique high-entropy token and a task-relevant framing.
- `CANARY_PUBLIC_NOTICE.md` states the regex and cites at least one precedent for the canary-filter practice.
- `private/canary_registry.yaml` has one entry per canary with all required fields, and the private-canary tokens do *not* appear in any public-facing artifact.
- `probe.py` implements at least two probes and runs end-to-end against a real (or specified) model with a documented seed.
- Positive control (Part E) demonstrates the probe fires on a known-contamination scenario. The demonstration is included in the writeup with the numbers.
- `PROBE_RUNBOOK.md` names a cadence, the models to probe, an escalation path, and a rotation rule.
- `CANARY_DESIGN.md` addresses all five required sections and explicitly discusses what canaries do not measure.

## Stretch goals

- **Multi-modality canaries.** For a multimodal benchmark, design canary items that combine a text token with an image feature (e.g. an image containing a QR code encoding the canary token). Probe by asking a multimodal model to describe or extract from the image.
- **Watermarked canaries.** Explore text-watermarking techniques (SynthID-Text or academic proposals for LLM watermarking) as a canary variant that survives paraphrase. Discuss the trade-offs versus the high-entropy token approach.
- **Canary-aware training filter.** Write a small utility that scans a text corpus for known public canary tokens and reports which benchmarks' canaries were found. This is the tool a responsible training operator would run on a corpus before pretraining.
- **Simulated leaderboard adversary.** Design a canary that would detect not just corpus inclusion but a leaderboard adversary that submits many probes against the private test set. Discuss what "canary" means in this adversarial setting and what shape the detector takes.
- **Extend to a multi-year rotation schedule.** Show what the canary set looks like after 8 rotations at 4 canaries added per rotation, including which old canaries are retired at each step and why.
