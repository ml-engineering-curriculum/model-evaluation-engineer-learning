# Public/Private Holdouts and Canary Sets

Chapter 5 measures contamination after the fact. Chapter 6 versions the benchmark so rotations are auditable. This chapter is the split-level design that gives both something to protect: a public/private holdout structure that reveals leakage by divergence, and a canary set engineered so that if the benchmark items ever enter a future training corpus, you can *prove* it.

Together they turn contamination from an unknowable property into an observable one. Once a canary hit fires, you know the item is in the corpus; once the public/private gap opens, you know a model has adapted to the public split. Both are early-warning systems designed at benchmark-authoring time — retrofitting them after release almost never works.

## Public vs. private holdouts: the mechanics

The split structure from Chapter 2 assumes three distinct roles for the test data:

- **`public_test`.** Published with the benchmark. Everyone can see the items, run their model, and report a score. Assumed to enter future pretraining corpora on the timescale of one to two years.
- **`private_test`.** Held privately by the benchmark maintainer. Never published. Scored by (a) accepting model outputs from a submitter and computing scores server-side, or (b) accepting a model or API key and running the eval maintainer-side.
- **`canary`.** Small set, published as part of the benchmark artifact, but engineered so that presence in a corpus can be detected by simple search. Purpose is *detection*, not *scoring*. See later sections.

The reason to split public from private is that the two together form a leakage detector: on a clean benchmark evaluated on a clean model, public and private scores should agree within the width of a paired-comparison CI (mod-101 Chapter 5). When they diverge — public score high, private score meaningfully lower — the model has fit the public split (contamination, targeted fine-tuning, prompt-engineered against the leaderboard) rather than the underlying construct.

## Designing the two splits to be exchangeable

For the divergence signal to be trustworthy, the public and private splits must be drawn from the same distribution. If they are not, an observed gap could be attributed to distribution difference rather than adaptation. Concrete design rules:

- **Same sampling frame.** Same source corpora, same collection window, same inclusion/exclusion filters.
- **Same instruction version and adjudication protocol.** Chapters 3–4 apply equally; the private set is not a lower-effort second-class artifact.
- **Stratified assignment.** Split assignment stratifies on the slice columns (`metadata.domain`, `metadata.language`, `metadata.difficulty`) so per-slice counts are similar. Random unbalanced splits produce spurious gaps.
- **Sized for adequate paired power.** The gap signal is a paired proportion or paired mean comparison. Use the MDE table (mod-101 Chapter 5) to pick a size that can detect a gap you care about (e.g. 2 percentage points at 80% power).
- **Assignment seed logged.** Store the split seed in `splits.yaml`; anyone with the raw store can reproduce the assignment.

If the public/private assignment is stratified and the two splits are of the same size, the null of "no adaptation" translates to "public score − private score = 0" up to a paired-bootstrap CI on the difference (mod-101 Chapter 4). Any operationally material gap is a leakage flag.

## Operating the private test set

A private test set is only private if you keep it that way. Operational patterns:

- **Server-scored submissions.** Submitters upload outputs (not the model). The maintainer scores server-side and returns the aggregate number. Item-level scores may or may not be returned; not returning them limits the information a submitter can extract about the private set.
- **API-scored submissions.** Submitters give a running model endpoint or an API key. The maintainer runs the eval and returns the aggregate. Higher trust than raw outputs; some submitters cannot participate for policy reasons.
- **Rate-limited submissions.** Each submitter is limited to `k` submissions per month. Prevents optimization against the private set by repeated probing. `k = 1` or `k = 2` per month is common.
- **Prompt / model rotation.** For high-stakes benchmarks, rotate the private items on a cadence so that even accumulated leakage from repeated submissions has a bounded window in which to matter.
- **Chain of custody.** The private items live in an access-controlled artifact store; access is logged; the log is auditable. Anyone with access must sign a use agreement.

The failure mode is not adversarial; it is drift. A private set that a wide team can see becomes public in a year through casual disclosure. Access should be minimal and logged.

## What the public/private gap tells you

Interpret the gap conservatively. Not every divergence is contamination; not every match is a clean bill of health.

- **Consistent public > private across models.** The public split is either easier (design bug — regenerate the split) or has been targeted by many teams (leaderboard optimization pressure). Either way, the private number is the more trustworthy one.
- **Public > private for one model, matched for others.** That model has adapted specifically. Contamination, targeted fine-tuning, or a prompt-engineered pipeline. Investigate before publishing the model's public number.
- **Public ≈ private but both surprising against a baseline you know well.** The gap detector cannot tell you anything about a construct that both splits share. It is a *relative* leakage detector, not an absolute quality one.
- **Private > public.** Rare. Usually a design or sampling bug (the private set is easier than the public), occasionally a signal that the model refused unusually often on published items (safety-tuning against the public items). Investigate.

Report the gap alongside the primary numbers. A model card that says "public 74.2%, private 73.8% (diff CI [−0.9, +1.7])" is more informative than the public number alone.

## Canary sets: engineering detectable leakage

A canary is an item deliberately designed to be greppable in a future training corpus. If the canary phrase later appears in a model's outputs, or is detectable via the model-behavioral probes in Chapter 5, you have proof that the corpus consumed the benchmark. This is the only way to *demonstrate*, rather than merely infer, post-hoc training-set inclusion for closed models.

### The canonical canary design

- **Unique high-entropy strings.** Each canary item contains a distinguishing token — a randomly generated identifier, a made-up proper noun, an unusual date, a pseudo-URL — that is extremely unlikely to occur naturally in web text. UUIDs, base-32 nonces, or GUID-shaped strings work; make them long enough (24+ characters) that random collisions are effectively zero.
- **Structured for retrieval.** The token appears verbatim in the item's `input`, `reference`, or both. Include an unambiguous, distinctive surrounding context ("the special code XYZ-ABC-123 refers to …") so that grepping the corpus for the token surfaces the full item, not just the token.
- **Registered in a canary registry.** A private file maps each canary token to its `item_id`, creation date, and the benchmark version at introduction. The registry is what you query later to answer "is any of our canary content in this training corpus."
- **Never published in isolated form outside the benchmark.** The canary lives *inside* the shipped benchmark artifact. If you also post it in a wiki, a slide deck, or a blog post, you contaminate your own detector by giving crawlers a second path in.

BIG-bench (Srivastava et al. 2022) is the widely-cited example of a public benchmark that shipped a documented canary string with a request to training pipelines to filter it out; that pattern is now common enough that many pretraining pipelines look for known canary strings and exclude them, which is itself useful — it means the benchmark's items may still enter the corpus, but at least the ones marked as canary are recognized.

### Two canary variants worth knowing

- **Public canary.** The distinguishing string is published in the benchmark artifact. Pretraining pipelines can filter for it. Good-faith training operators exclude these items. Detects careless, not adversarial, inclusion.
- **Private canary.** The distinguishing string is registered privately and never published as a canary; the item appears in `public_test` looking like any other item. A future model that reproduces the item verbatim proves inclusion even against a training operator that filters the public canary list. Higher-signal, harder to run because the maintainer must periodically probe deployed models with the private canary token and check for reproduction.

Ship both. Public canaries are a courtesy; private canaries are the actual detector.

### Probing for canary presence

Two ways to query a deployed model for canary contamination:

- **Verbatim continuation probe.** Prompt the model with the first `k` tokens of the canary item (up through the distinguishing token) and check whether the model completes it verbatim. High-signal but only fires if the corpus contained the item literally.
- **Log-likelihood probe.** Compute the model's log-likelihood on the canary item and compare against a matched control (a non-canary item with similar length and topic, drawn from data collected after the model's training cutoff). A sharp likelihood gap on canary vs. matched control is a memorization signature (Chapter 5's probe family, but with a much cleaner control by construction).

A quarterly probe against every model you evaluate produces a running record of canary hits. When the record shows a hit, you have proof, not inference, that the benchmark items entered that model's training corpus.

### What canaries do not measure

Canaries are pointwise proofs of inclusion, not distribution-level estimates of contamination. A single canary hit says "at least one item from this benchmark was in the training corpus." It does not tell you what fraction of the benchmark's items are contaminated, or which ones. Use canaries alongside — not instead of — the surface, semantic, and behavioral detectors of Chapter 5.

They also do not detect *paraphrased* inclusion. A training pipeline that ingests a rewritten version of the item will not trigger a canary token match. This is why the private canary must be a token that is extremely hard to rewrite out of the item (embedded in a structural way, not merely mentioned in passing).

### Canary set size and refresh

- **Size.** 50–200 items is typical. Too few and probes are underpowered; too many and canaries dilute the benchmark's utility for scoring.
- **Refresh.** Add a small number of new canaries with each MINOR release (Chapter 6). The old canaries stay in place for continued monitoring; the new ones extend the observability window forward.
- **Retire on confirmed hit.** Once a canary hit is observed for a given model, that specific canary has done its job for that model. Keep it in the registry for the audit trail; add a replacement.

## Combining public, private, and canary into one release

For a fresh benchmark release, the split shape typically looks like:

- `dev`: 500 items, published. Small enough to give a signal, not big enough to be a headline.
- `public_test`: 2000 items, published. The headline number.
- `private_test`: 2000 items, held privately, stratified to match `public_test`. Divergence detector.
- `canary`: 100 items, of which 50 are public canaries (marked and filterable) and 50 are private canaries (indistinguishable from other items but registered).

The manifest lists the counts and the split hashes. The private test set and the private canary registry live in an access-controlled artifact store; only the manifest hashes appear in the public repo.

## When the detectors fire

Playbook when either signal trips:

- **Public/private gap opens for a specific model.** Investigate: rerun with a different sampling seed to rule out variance; check the model's release notes and any published training data cutoff; run Chapter 5's behavioral probes against the model on the flagged split. If the gap survives, publish the finding (transparency wins trust with benchmark consumers) and consider rotating the public split.
- **Canary hit fires.** Publish that a hit was observed with the affected model and date. Rotate the affected public canaries. If the model in question is one you evaluate regularly, add its post-cutoff traffic to the sourcing pool for a fresh canary batch.
- **Both fire on the same model in the same cycle.** The benchmark has been compromised for this model. Rotate items (Chapter 6 MINOR bump), consider the benchmark's remaining useful life, and reference the deprecation policy.

## What holdouts and canaries do not fix

- **The construct still has to be right.** No amount of split hygiene fixes a benchmark that operationalizes the wrong construct (mod-101 Chapter 2). Chapters 3–4 remain load-bearing.
- **Small samples are still small.** A private test set of 200 items has a wide CI regardless of how well-kept it is. Size for power; use paired comparisons.
- **Adversarial submitters exist.** A determined actor can extract information about a private test by probing repeatedly. Rate limits and periodic rotation slow the extraction; they do not eliminate it. High-stakes benchmarks that face adversarial pressure (leaderboards with prize money, regulatory certification) need stronger designs than this chapter covers.

## Summary

Public/private holdouts and canary sets are the split-level defenses that make contamination observable. Public and private test splits, drawn from the same distribution and scored on the same items, expose per-model adaptation as a divergence between the two scores. Canary items — engineered with distinguishing high-entropy tokens, registered privately, probed periodically — turn future training-set inclusion from an inference into an observation. Ship all three (dev / public / private / canary) at benchmark v1, keep the private set access-controlled and rate-limited, refresh canaries at each MINOR bump, and read divergences and canary hits as first-class alerts that feed back into the rotation and deprecation policies of Chapter 6.
