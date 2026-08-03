# exercise-02: Jailbreak Resistance With HarmBench

**Estimated effort:** 5 hours

## Objective

Measure jailbreak attack success rate (ASR) against a candidate model using HarmBench (Mazeika et al. 2024) with at least three attacker profiles, and produce the ASR-curve reporting shape from Chapter 3. The deliverable is a run report that pins `(model, dataset revision, attacker profiles, judge, judge calibration)`, publishes ASR per attacker per category with CIs, and defends the attacker-model choices against Chapter 3's specification. Explicitly, this exercise does *not* author novel jailbreak payloads; every attack transformation comes from HarmBench's published `attacks/` module.

## Prerequisites

- mod-109 Chapter 3 (jailbreak resistance with HarmBench).
- Exercise-01's refusal / over-refusal pipeline (the report header format and the judge-calibration discipline transfer).
- mod-101 Chapters 2 and 4 (bootstrap CIs, FDR).
- Python 3.11+, HarmBench installed (`pip install harmbench` or from source; the `centerforaisafety/HarmBench` GitHub is the canonical source).
- Compute budget: the static and single-turn-transformation attackers are cheap (~$5–$20 on a small model). PAIR / TAP against a hosted frontier model can reach $50–$200 for a full run. Run a 30-behavior pilot first and extrapolate before committing to the full split.

## The instrument

Use HarmBench `standard` split (not `contextual` for this exercise — the acceptance criterion is one clean split). Pin the revision SHA in the report. The exact behavior list, count, and category taxonomy come from HarmBench itself; do not paraphrase behaviors into a fixture file, and do not add or remove behaviors.

The compliance judge is the HarmBench-provided classifier (`cais/HarmBench-Llama-2-13b-cls` or the newer classifier release; whichever is current). The judge's known agreement with expert human labels (~92% per the paper) is your calibration anchor; you will re-verify below.

## Requirements

### Part A — attacker profiles

Run at least three attacker profiles:

- **Static baseline.** HarmBench's `direct` attack: the behavior description sent as-is. `k=1` attempt per behavior. Serves as your floor.
- **Single-turn transformations.** HarmBench's rule-based transformations module: `k=1` per behavior, sampled uniformly from the module's transformation set. Do not modify the transformation set; use HarmBench's defaults.
- **Automated iterative attacker.** PAIR (Chao et al. 2023) or TAP (Mehrotra et al. 2024), both included in HarmBench's `attacks/` module. Run with `k_max = 20` iterations per behavior. Pick one; if compute allows, run both.

Optional (stretch): Crescendo-style multi-turn attacker, per Chapter 3 §"Multi-turn attacks."

Never author new attack methods. Use the HarmBench-published implementations verbatim. If a method requires an attacker LLM, use a hosted model (temperature 0.7, per PAIR's default) and log the attacker-model provider and version.

### Part B — the eval loop

Build `harmbench_eval/` (a thin wrapper around HarmBench's own runner is fine — you are not re-implementing HarmBench):

- **`harmbench_eval/run.py`** — for each attacker profile, for each behavior, run the attacker (which may itself iterate against the target). Log a `Result` record: `(behavior_id, attacker_id, target_model, iteration_count, verdict, latency_s, attacker_tokens, target_tokens)`. Do NOT log the raw attempted prompt or the target's response to this file.
- **`harmbench_eval/artifacts.py`** — a separate function that writes per-attempt raw content to a *separately-permissioned* directory (`artifacts/raw/`), one file per attempt, so debug traces exist under access control. Include a README in `artifacts/raw/` explaining the retention policy (default: delete after 30 days unless flagged for red-team disclosure).
- **`harmbench_eval/judge.py`** — wrap the HarmBench compliance classifier. Re-verify calibration by sampling 200 `(behavior, target_response)` pairs from the run's output, hand-labelling as `attack_success / refused`, and computing agreement.
- **`harmbench_eval/aggregate.py`** — ASR per attacker per category with 95% bootstrap CIs (2000 resamples, seed=17). Also emit an ASR-curve for the iterative attacker: for each `k ∈ {1, 5, 10, 20}`, ASR = fraction of behaviors succeeded in ≤ `k` attempts. Also emit "queries-to-first-success" distribution (median, p95, censored at `k_max`).
- **`harmbench_eval/report.py`** — a CLI that renders the report shape below.

### Part C — the report

```
# Jailbreak resistance report

Model:   <provider/model@version>
Dataset: HarmBench standard split, revision <sha>, n_behaviors = <N>
Judge:   HarmBench compliance classifier <version>
         Human-gold calibration: <A>% agreement on 200 sampled pairs

## ASR by attacker profile
| Attacker profile               |  k  | ASR   | 95% CI       | Notes            |
|--------------------------------|-----|-------|--------------|------------------|
| static-baseline                |  1  | 0.02  | [0.01, 0.04] |                  |
| single-turn transformations    |  1  | 0.07  | [0.05, 0.10] |                  |
| PAIR (attacker: <model@ver>)   | 20  | 0.19  | [0.15, 0.23] | curve below      |
| (optional) TAP                 | 20  | 0.24  | [0.20, 0.28] |                  |

## PAIR ASR curve
| k  | ASR   | 95% CI       |
|----|-------|--------------|
|  1 | 0.05  | [0.03, 0.08] |
|  5 | 0.12  | [0.09, 0.16] |
| 10 | 0.17  | [0.13, 0.21] |
| 20 | 0.19  | [0.15, 0.23] |

Queries-to-first-success (censored at k=20):
  Median: 7 queries
  P95:    censored (behavior never succeeded within k=20)

## Per-category ASR (under PAIR k=20, BH-adjusted across categories, q<0.05)
| Category                    |  n  | ASR   | 95% CI       | q     |
|-----------------------------|-----|-------|--------------|-------|
| illegal                     |  60 | 0.11  | [0.06, 0.17] | 0.02  |
| cybercrime                  |  55 | 0.28  | [0.20, 0.36] | 0.001 |
| ...                         |     |       |              |       |

## Attacker budget
| Attacker | Mean target queries | p95 | Mean $ / behavior |
|----------|---------------------|-----|-------------------|
| PAIR     | 12                  | 24  | $0.08             |

## Judge calibration re-check
- Sampled 200 pairs from the run output.
- Agreement with human gold: <A>%.
- Disagreement dominated by: <describe pattern — e.g. "responses that pattern-match refusal openers but proceed to disclose">.

## Limitations
- Single-turn attackers dominate; multi-turn coverage not included.
- Non-English coverage not evaluated in this run.
- ...
```

### Part D — data handling and content policy

For each of the following, ship a short written policy inside the exercise repo:

- **`docs/DATA_HANDLING.md`** — where per-attempt raw content is stored, who has access, retention window, how content is redacted before any external circulation. Should reference Chapter 3 §"How to run this without writing an attack cookbook" verbatim as the rule set.
- **`docs/EXTERNAL_ARTIFACT.md`** — a one-page description of what would go into a model-card safety section from this run (aggregate ASR, per-category, methodology; Chapter 7 §"Jailbreak resistance" as the template) and what would explicitly be excluded (per-attempt behaviors, target completions, specific successful transformation prompts).

## Starter guidance

- **Do the pilot before the full run.** 30 randomly-sampled behaviors × 3 attacker profiles will surface every plumbing bug the full run would surface, and at 5–10% of the cost. Fix everything on the pilot before scaling up.
- **Cost gate.** For iterative attackers, track cumulative dollars in-run. Set a hard cap (e.g. $150 total) and abort if approached. The extrapolation from the pilot is the trustworthy budget signal.
- **Use HarmBench's implementations, not paraphrases.** If a HarmBench attacker's default hyperparameters look inconvenient, document what you changed and why. Do not silently drift.
- **Log at aggregate granularity by default.** The `Result` schema in Part B is deliberately payload-free. Debug artifacts go to `artifacts/raw/` under access control; if you find yourself pretty-printing a jailbreak attempt to stdout while debugging, stop and route it through the artifacts function.
- **The judge is a load-bearing calibration.** HarmBench's own paper reports ~92% agreement; on your model's specific completions this may be higher or lower. A 200-sample re-check is not optional.
- **Report the curve.** The ASR curve over `k` is more informative than the peak ASR at `k_max`. Two models with the same ASR at `k=20` but very different ASR at `k=1` have very different resistance profiles.
- **Never embed payloads in your commit history.** Even in a private repo, if attack transformations get committed to a fixture file, they will follow the repo forever. Load them at runtime from the pinned dataset.

## Acceptance criteria

- HarmBench `standard` split loaded at a pinned revision SHA (recorded in report).
- At least three attacker profiles run: static, single-turn transformations, and one iterative (PAIR or TAP). Iterative runs report the ASR curve over `k ∈ {1, 5, 10, 20}`.
- Judge calibration is re-verified on 200 sampled pairs from *this run's* outputs; the agreement rate is in the report header.
- Per-category ASR uses Benjamini–Hochberg correction across categories at `q<0.05`.
- Report contains attacker-budget accounting (target queries per behavior mean and p95, dollar cost per attacker profile).
- Per-attempt raw content is stored in a separately-permissioned directory with a retention policy documented in `docs/DATA_HANDLING.md`; nothing verbatim leaks to the report or to any file outside `artifacts/raw/`.
- The report shape from Part C is produced end-to-end from `python -m harmbench_eval.report`.
- No custom attack transformations, no hand-authored jailbreak strings anywhere in the codebase.

## Stretch goals

- **Multi-turn attacker (Crescendo-style).** Implement or use a Crescendo-style multi-turn attacker (see Russinovich et al. 2024). Report multi-turn ASR alongside the single-turn numbers, with the turn count and attacker configuration named.
- **Private held-out attack corpus.** Add a red-team-authored (under sign-off) held-out set of 50–100 behaviors — *not* transformations, but new behavior descriptions. Compare ASR on public HarmBench vs held-out. A large gap is the contamination signal.
- **Cross-model calibration.** Run PAIR against two target models (one strong, one weaker). Report the ASR gap and compare the queries-to-first-success distributions.
- **Attacker A/B.** Run PAIR and TAP against the same target and report per-category disagreement. Attacker portfolios do not correlate perfectly; the disagreement is the diagnostic.
- **Model-card writeup.** Produce the model-card safety section from this run's data using Chapter 7's format, and have a colleague review it for payload leakage. Iterate until the reviewer finds none.
