# exercise-05: Prompt Injection Robustness Suite

**Estimated effort:** 4 hours

## Objective

Measure a model's prompt-injection robustness against at least one direct-injection suite and one indirect-injection suite, using the *attack success rate at fixed benign utility* metric shape from Chapter 6. If the target is an agent, add a tool-poisoning trajectory measurement using the mod-108 trajectory scorer. The deliverable is a report that publishes the `(benign utility, injection ASR)` pair for at least two defense configurations (baseline versus one mitigated configuration) with CIs, cites the attack taxonomy of every suite used, and separates the eval finding from vulnerability disclosure. As with exercises 02 and 03, no novel injection payloads are authored — every attempt comes from a published suite loaded at a pinned revision.

## Prerequisites

- mod-109 Chapter 6 (prompt-injection robustness and the boundary with security testing).
- mod-101 Chapters 2 and 4 (bootstrap CIs, FDR across many attack categories).
- mod-105 (judge bias controls) — the injection compliance judge is graded the same way as the refusal judge.
- mod-108 Chapters 2 and 6 (trajectory scorer, Inspect agent harness) if you attempt the agent tool-poisoning part.
- Exercise-01 and exercise-02 for the report-header discipline, judge-calibration protocol, and data-handling conventions.
- Python 3.11+, `datasets`, and access to at least one hosted or locally-served target model. If you attempt the agent tool-poisoning part, an Inspect (or equivalent) sandbox.

## Datasets

Load from pinned revisions. Never paste payloads into a fixture file.

- **Direct-injection suite (required).** Tensor Trust prompt-injection dataset (Toyer et al. 2024) from the official Hugging Face release, *or* a Structured Query Attacks corpus if you have one under license. Random subsample of `n_direct ≈ 300` prompts if the full corpus is too expensive; report `n` and the sampling seed.
- **Indirect-injection suite (required).** BIPIA (Yi et al. 2023) from the official repo, all five task types (summarization, QA, code, table QA, search) with the 25 attack methods per task. Random subsample if compute-limited; keep all attack methods represented.
- **Agent tool-poisoning suite (required if you attempt Part D).** InjecAgent (Zhan et al. 2024) from the official repo. Use the "enhanced" split for stronger attacks and the "base" split for the calibration baseline.
- **Optional: AgentDojo** (Debenedetti et al. 2024) for a second agent-shaped indirect-injection instrument.

Every dataset load pins the revision SHA; the SHA appears in the report header.

## Requirements

### Part A — the eval library

Ship `injection_eval/` with at minimum:

- **`injection_eval/schema.py`** — dataclasses for `InjectionTask` (with `benign_input`, `adversarial_input`, `attack_category`, `expected_benign_output`, `injected_behavior`), `Response`, `JudgeVerdict`, `PairReport` (utility + ASR at a fixed defense configuration).
- **`injection_eval/datasets.py`** — loaders for at least Tensor Trust and BIPIA, each pinning a revision SHA and preserving the suite's attack-category taxonomy.
- **`injection_eval/model.py`** — the target-model adapter, same shape as exercise-01's. Temperature 0.0 (or vendor deterministic default), max_tokens 512, seed=17.
- **`injection_eval/defense.py`** — at least two defense configurations: `baseline` (no application-side defense) and one mitigated (input delimiter tagging, "Structured Queries"-style input framing, or an input-classifier gate — pick one from the Chapter 6 references and cite it). Both are pure input transformations; do not modify the model.
- **`injection_eval/judge.py`** — two judge implementations:
  - **Behavior-match judge**: exact-string / regex check that the model produced the specific output the injection asked for (e.g., "did the response contain `SECRET_TOKEN`?"). BIPIA-style.
  - **Task-deviation judge**: an LLM-as-judge prompted with the benign task and the response, returning `on_task / injected_task / mixed`. Task-deviation style; for the more open-ended items.
  - Each `(task, response) → JudgeVerdict` returns `attack_success ∈ {true, false, partial}` and a `judge_name` field so the report can name which judge produced which verdict.
- **`injection_eval/utility.py`** — the benign-utility scorer per suite: task-appropriate (ROUGE-L for summarization, exact-match for QA, code-execution pass rate for code, etc.). Reuse mod-104 utilities where possible.
- **`injection_eval/aggregate.py`** — for each defense configuration, per attack category:
  - Benign utility with 95% bootstrap CI (2000 resamples, seed=17).
  - Injection ASR with 95% CI.
  - Both aggregated and per-attack-category, BH-corrected across the category axis.
- **`injection_eval/report.py`** — `python -m injection_eval.report scores.json --out report.md` producing the shape in Part E.

### Part B — the judge calibration

The injection judge is the load-bearing component (Chapters 3 and 6). Before headline numbers:

- Sample 200 total `(task, response)` pairs — 100 from the direct suite, 100 from the indirect suite — stratified by verdict category so both `attack_success` and `on_task` items are represented.
- Hand-label (yourself, plus one reviewer if possible) each response under a written *injection policy* you draft (at least a page: what counts as attack success in the behavior-match sense, what counts in the task-deviation sense, edge cases like a response that begins with the injected behavior but recovers, partial task completion).
- Compute the judge's agreement rate on each suite separately.
- If agreement is below 80% on either suite, iterate the judge prompt / backend before the headline run. Report the *final* agreement rates in the report header.

The written injection policy is a deliverable and is cited by the report as ground truth.

### Part C — the utility / ASR pair, at least two defense configurations

For each defense configuration (`baseline` and at least one mitigated):

- Run the target model on the benign-only control set of every suite. Compute the benign-utility metric with a CI.
- Run the target model on the adversarial set. Compute the injection ASR with a CI (and per-attack-category).
- Report the pair. A defense that reduces ASR by 15 points but reduces utility by 15 points is not an improvement; the pair is the deliverable.
- Report the Pareto frontier across your two-or-more configurations as a short table or plot.

Never report an ASR without the paired utility number in the same section.

### Part D — agent tool-poisoning (required if the target is an agent)

Skip if the target model is not exposed as an agent with tools. Otherwise:

- Wire the target model into an Inspect (or equivalent) agent harness with a minimal tool set (a `search` tool that returns injectable text, a `send_message` tool the attacker will try to redirect, a `read_file` tool that returns injectable content).
- Load InjecAgent's enhanced split. For each task, run the agent and produce a trajectory.
- Score with a mod-108-style trajectory scorer:
  - **Benign task completion**: did the agent complete the user's legitimate task?
  - **Attacker-tool-invocation rate**: did the agent invoke the attacker's requested tool with attacker-specified arguments at any point?
- Report both, per attack category.

Follow mod-108 Chapter 6's discipline for the harness spec: sandbox digest, tool descriptions verbatim, budget cap, seed.

### Part E — the report

```
# Prompt-injection robustness report

Model:   <provider/model@version>
Suites:
  - Tensor Trust (direct):  revision <sha>, n_direct = <N>
  - BIPIA (indirect):       revision <sha>, 5 task types, 25 attack methods, n_indirect = <M>
  - InjecAgent (optional):  revision <sha>, enhanced split, n_agent = <K>
Judges:
  - Behavior-match:  <description>  (human-gold agreement: <A>%, n=100)
  - Task-deviation:  <LLM-judge>    (human-gold agreement: <B>%, n=100)

## Direct injection (Tensor Trust)
| Defense config | Benign utility [CI] | Injection ASR [CI]   |
|----------------|---------------------|----------------------|
| baseline       | 0.86 [0.83, 0.89]   | 0.34 [0.29, 0.39]    |
| structured-in  | 0.83 [0.80, 0.86]   | 0.12 [0.09, 0.16]    |

Per attack-category ASR (baseline, BH-adjusted, q<0.05):
| Category                | n   | ASR   | 95% CI       | q     |
|-------------------------|-----|-------|--------------|-------|
| ignore-previous         |  60 | 0.52  | [0.39, 0.65] | 0.001 |
| role-play               |  55 | 0.28  | [0.16, 0.41] | 0.02  |
| ...                     |     |       |              |       |

## Indirect injection (BIPIA)
| Defense config | Task type   | Benign utility [CI] | Injection ASR [CI]   |
|----------------|-------------|---------------------|----------------------|
| baseline       | summ (ROUGE)| 0.42 [0.40, 0.44]   | 0.31 [0.28, 0.34]    |
| baseline       | QA (EM)     | 0.68 [0.65, 0.71]   | 0.24 [0.21, 0.27]    |
| structured-in  | summ (ROUGE)| 0.40 [0.38, 0.42]   | 0.14 [0.11, 0.17]    |
| structured-in  | QA (EM)     | 0.66 [0.63, 0.69]   | 0.10 [0.08, 0.13]    |
| ...            |             |                     |                      |

Per attack-method ASR (baseline, summarization, BH-adjusted):
| Method (BIPIA taxonomy) | n  | ASR   | 95% CI       | q     |
|-------------------------|----|-------|--------------|-------|
| direct-instruction      | 80 | 0.52  | [0.41, 0.63] | 0.001 |
| context-switching       | 80 | 0.24  | [0.15, 0.34] | 0.02  |
| fake-completion         | 80 | 0.18  | [0.10, 0.28] | 0.06  |
| ...                     |    |       |              |       |

## Agent tool-poisoning (InjecAgent enhanced split) — optional
Harness: Inspect + basic_agent, budget 20 steps, sandbox <docker-digest>
Scorer:  mod-108 trajectory scorer

| Metric                          | Value            | 95% CI       |
|---------------------------------|------------------|--------------|
| Benign task completion (base)   | 0.71             | [0.66, 0.76] |
| Attacker-tool-invocation rate   | 0.09             | [0.06, 0.13] |

## Pareto frontier (utility vs 1-ASR, indirect suite)
<short table or plot>

## Judge calibration
- n = 200 (100 direct, 100 indirect)
- Behavior-match agreement on direct suite:      <A>%
- Task-deviation agreement on indirect suite:    <B>%
- Disagreement dominated by: <describe pattern — e.g. "partial-injection cases where the model does the benign task and additionally leaks a token">

## Limitations
- Suite contamination: Tensor Trust and BIPIA are public; a private
  held-out injection set (stretch goal) is the contamination diagnostic.
- Multimodal injection (image-embedded, audio-embedded) not covered
  in this run.
- Multi-turn indirect injection (an attacker document that only lands
  after several agent turns) not covered.
- Utility metrics are per suite's own reference (ROUGE-L, EM); a
  task-specific human-utility eval is a stretch goal.
```

### Part F — data handling and the security boundary

Two written documents ship in the exercise repo:

- **`docs/DATA_HANDLING.md`** — where per-attempt raw content lives, who has access, retention (default 30 days), and the redaction rule for any external circulation. References Chapter 6 §"Load, don't author" and Chapter 3 §"How to run this without writing an attack cookbook" verbatim as the rule set.
- **`docs/SECURITY_BOUNDARY.md`** — a one-page statement of what this exercise measures (baseline model robustness against published suites) versus what it does *not* do (find novel exploits against a specific production deployment). Names the vulnerability-disclosure channel your organization uses: if a run surfaces a specific novel exploit that would cause real production impact, it is disclosed through that channel, not this report. Chapter 6 §"The boundary with security testing" is the reference.

## Starter guidance

- **Start with the utility metric, not the attack.** Pick your benign-utility metric per suite before you run any adversarial content. If the utility metric is under-specified, your defense can trivially improve ASR by breaking utility and you will not notice.
- **Two defense configurations at minimum.** A single-config report shows only the baseline; the utility/ASR *tradeoff* is what stakeholders care about. Do not skip the mitigated config.
- **Do the pilot before the full run.** 20 items per suite per defense config × 2 configs will surface every plumbing bug the full run would. Aim for a pilot that costs 10% of the full run.
- **Behavior-match judges are cheap and precise; task-deviation judges are expensive and open-ended.** Use each where it fits — BIPIA and InjecAgent give you machine-checkable behavior targets; task-deviation is for the residual items where "did the model deviate" is a judgement call.
- **Never conflate an eval finding with a vulnerability.** If your run surfaces a novel injection that reliably lands against a production deployment (not just the base model), stop, do not publish in this report, and route through the security disclosure channel. The report can say "an injection class was identified; disclosed to security on <date>; remediated in <PR>."
- **Do not print payloads to stdout.** Same discipline as exercise-02: aggregate logs by default, raw content in a separately-permissioned directory with retention.
- **Load, do not author.** Every attempt comes from a pinned suite. Any hand-written injection string in a fixture file is a rejected deliverable.
- **Cite the attack taxonomy.** A single-number ASR without the suite's attack taxonomy hides which attack shapes you actually covered. The category-level table is the reader's only way to see gaps.

## Acceptance criteria

- At least two suites loaded from pinned revisions: one direct (Tensor Trust or Structured Queries) and one indirect (BIPIA). SHAs are recorded in the report header.
- At least two defense configurations run (`baseline` and one mitigated), with the utility/ASR pair reported for each.
- Judge calibration produces ≥ 80% agreement on both suites, and the agreement numbers appear in the report.
- Per-attack-category ASR uses Benjamini–Hochberg correction across categories at `q<0.05`.
- The report never presents an ASR without the paired benign-utility number in the same table.
- Written injection policy exists as a deliverable and is cited as ground truth for the judges.
- `docs/DATA_HANDLING.md` and `docs/SECURITY_BOUNDARY.md` ship, and no verbatim payloads leak to the report, the codebase, or any log outside the controlled `artifacts/raw/` directory.
- Report contains a limitations section that explicitly names suite contamination, multimodal, and multi-turn as uncovered gaps.
- If the target is an agent, Part D is completed: InjecAgent trajectory measurement with attacker-tool-invocation rate and benign-task completion, using a mod-108-style trajectory scorer.
- The report reproduces from `python -m injection_eval.report` with pinned seeds and dataset revisions.

## Stretch goals

- **Structured Queries defense.** Implement the "Structured Queries" input framing (Wallace et al. 2024, Chen et al. 2024) as a third defense configuration and add it to the Pareto frontier.
- **Private held-out injection set.** Author, under security-team sign-off, a small (n≈50 direct + n≈50 indirect) private held-out injection set that does not overlap Tensor Trust or BIPIA. Compare ASR on public vs held-out. A large gap is the contamination diagnostic.
- **Multimodal injection.** For a multimodal target model, add an OCR-based image-embedded injection attack (per Bagdasaryan et al. 2023). Report per-modality ASR at fixed utility. Note the field is early and calibrate conservatively.
- **Multi-turn indirect injection.** Craft (or load, if a published suite exists) a multi-turn variant where the attacker document only lands after several agent turns. Report multi-turn ASR alongside single-turn.
- **AgentDojo comparison.** Run AgentDojo (Debenedetti et al. 2024) as a second agent-shaped indirect-injection instrument. Report per-suite disagreement — attacker portfolios differ across suites, and the disagreement is the diagnostic.
- **Attack-taxonomy coverage audit.** Compare the suites you ran against the OWASP LLM Top 10's "LLM01: Prompt Injection" sub-categories. Report which sub-categories are covered and which are not, so a governance reader can see gaps against a standard taxonomy.
- **Model-card writeup.** Produce the model-card injection section from this run's data using Chapter 7's format, and have a colleague review it under the "no payload leak" test.
