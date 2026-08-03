# exercise-02: SWE-bench Reproduction

**Estimated effort:** 6 hours (mostly wall-clock on image builds and inference)

## Objective

Run SWE-bench Verified (or SWE-bench Lite) against a candidate model with a documented agent harness, produce the `% Resolved` and `% Applied` numbers, and reproduce a published number *within tolerance* — or explain, with evidence, why you cannot. The deliverable is a reproducibility bundle: predictions file, evaluation report JSON, a written gap-analysis, and the manifest that pins the tuple `(model, harness, prompt, budget, dataset)`. This is where the Chapter 3 reproduction discipline gets tested against a real published number.

## Prerequisites

- mod-108 Chapter 3 (SWE-bench reproduction end-to-end).
- Exercise 01's trajectory scorer library (you will invoke it on the agent traces if your harness emits them).
- Docker with **at least 100 GB of free disk** for the instance images (SWE-bench Verified is ~500 instances × 1–3 GB each; the base + environment layers are large caches).
- A machine with ≥16 GB RAM and ≥8 CPU cores. A workstation is fine; a small cloud VM works if disk is provisioned appropriately.
- Access to at least one API-hosted model or a self-hosted OSS model served via vLLM. Budget for the run: a modest agent harness against `gpt-4o` or `claude-3-5-sonnet` on Verified is typically **$100–$600** depending on max-turns and retry policy. Run 10 instances first and extrapolate before committing.

## The benchmark

Use **SWE-bench Verified** (500 instances, human-verified for spec clarity and test determinism). If you cannot afford Verified end-to-end, use **SWE-bench Lite** (300 instances, easier); note the split at the top of the report. Do not use the full 2,294-instance split — the noise floor from flaky tests and ambiguous specs will dominate your gap analysis.

Load via Hugging Face; **pin the revision**:

```python
from datasets import load_dataset
ds = load_dataset("princeton-nlp/SWE-bench_Verified", split="test", revision="<sha>")
```

The revision SHA belongs in the manifest. Do not use a bare load; the Verified split has been re-released and future re-releases will silently change the numerator of your metric.

## Requirements

### Part A — pick a harness and a comparability anchor

Pick one of:

- **SWE-agent** (`github.com/princeton-nlp/SWE-agent`). The reference agent from the SWE-bench authors. Has published numbers against many models.
- **OpenHands** (formerly OpenDevin, `github.com/All-Hands-AI/OpenHands`). A widely-benchmarked agent stack with recent public SWE-bench Verified numbers.
- **Aider** (`github.com/paul-gauthier/aider`). Simpler agent than the above two; not always benchmark-competitive but easy to run and easy to reason about.
- **An Inspect-based harness** (mod-104 Chapter 5, plus mod-108 Chapter 6). Correct for internal builds; you will not reproduce a *published* number with a from-scratch Inspect harness, so if you pick this route your comparability claim is against your own baseline runs, not a paper.

Then pick a **comparability anchor**: a specific published `(model, harness, split)` triple with a public number you will target. Examples (all subject to leaderboard drift — verify before starting):

- OpenHands + `gpt-4o` on Verified.
- SWE-agent + `claude-3-5-sonnet` on Verified.
- Aider + `claude-3-opus` on Lite.

Document the anchor and the source (paper, GitHub README, leaderboard entry with date) at the top of your report. The anchor number is what "reproduction within tolerance" is measured against.

### Part B — build the images

Follow the harness's install steps to build the base, environment, and instance images. Expect this to take **hours** on first run; cache aggressively. Verify that at least three instances pass their `PASS_TO_PASS` tests *before* you apply any model patch (a sanity check that the environment is healthy):

```bash
python -m swebench.harness.run_evaluation \
  --dataset_name princeton-nlp/SWE-bench_Verified \
  --predictions_path fixtures/gold_patches.jsonl \
  --run_id sanity_gold \
  --report_dir reports/sanity/
```

`gold_patches.jsonl` should be the human-authored `patch` field from the dataset (which is the reference fix). If gold patches don't `resolve` at ≥ 95%, your environment is broken; fix before running the model.

### Part C — generate predictions

Configure the harness with the model and a documented budget:

- `max_turns` (aka `max_iters` in some harnesses): default 40 unless the harness recommends otherwise.
- Temperature: 0.0–0.2 for reproducibility; higher only if you're intentionally doing `pass^k`.
- Any tool-related settings that materially affect behavior (retrieval strategy, file-view chunking, patch format).

Run the agent over all instances (or a subset if cost-limited — see the truncation note below). Emit a `preds.jsonl` in the SWE-bench format:

```jsonl
{"instance_id": "django__django-11133", "model_name_or_path": "gpt-4o-2024-08-06+swe-agent-0.7.0", "model_patch": "diff --git ..."}
```

If your harness emits full agent traces (Inspect eval logs, SWE-agent trajectory JSON), keep them — you will use them in Part E.

**Cost gate.** Before running the full split, run 10 randomly-sampled instances end-to-end and record actual dollar cost. Extrapolate. If the extrapolated bill exceeds your budget, either use Lite instead or run a Verified subset — 200 instances is often enough to establish a reproduction within noise and is a defensible size to publish, as long as you're explicit about it.

### Part D — grade

Run the SWE-bench evaluator against your predictions:

```bash
python -m swebench.harness.run_evaluation \
  --dataset_name princeton-nlp/SWE-bench_Verified \
  --predictions_path preds.jsonl \
  --max_workers 8 \
  --run_id my_run_2026_08_10 \
  --report_dir reports/my_run/
```

The report JSON contains per-instance verdicts. Compute:

- **% Resolved** = `resolved / total`
- **% Applied** = `(resolved + applied_but_unresolved) / total`
- **% Environment-error** = `errored / total` (report separately; do not count against the model)
- **Per-repo breakdown** of `% Resolved` (12 repos for Verified). Some repos will be systematically harder or easier for the model; the breakdown is the diagnostic.

### Part E — trajectory-level analysis (if your harness emits traces)

If your harness emits agent traces, adapt them into your exercise-01 canonical schema and run the trajectory scorer over the resolved and unresolved cohorts separately. Report:

- Mean tool calls per trajectory, per cohort (resolved / applied_but_unresolved / patch_apply_failed).
- Redundant-call rate, per cohort.
- Termination-reason distribution (submitted vs budget_exhausted vs harness_error).
- Cost distribution (median, p95) per cohort in dollars.

The specific claim you're checking: **do failed trajectories cost more than successful ones?** In most published SWE-bench runs they do — the model thrashes on hard cases. Report the ratio.

### Part F — the gap analysis

Ship `REPORT.md` (2–4 pages) covering:

1. **Setup.** Model + version, harness + version, dataset revision, budget caps, sandbox / Docker configuration, machine spec.
2. **Anchor.** The specific published `(model, harness, split)` triple you targeted, with a link to the source and a date.
3. **Numbers.** Your `% Resolved`, `% Applied`, `% Environment-error`, per-repo breakdown, and the gap versus the anchor (positive or negative, in points).
4. **Gap analysis.** For any gap ≥ 3 points, walk the four sources from Chapter 3 (different harness, prompt sensitivity, sampling variance, dataset drift) and attribute the gap to one or more. If the gap is < 3 points, note it as within-noise and move on.
5. **Trajectory-level slice (Part E) if applicable.**
6. **Reproducibility manifest** (see Part G).

### Part G — the reproducibility manifest

Ship `MANIFEST.md` pinning at minimum:

- Model provider, model ID, version string, temperature, top-p, seed if honored, `max_tokens`.
- Agent harness name, version (git SHA), any local patches applied, system prompt file (linked, not paraphrased), tool descriptions.
- Budget caps: `max_turns`, `max_tokens_per_turn`, `max_wall_clock_s`.
- Dataset name, revision SHA.
- `swebench` package version, Docker version, base-image digests, evaluator config.
- Test runner (`pytest` version), per-instance timeout.
- Run identifier, machine spec, run start/end timestamps.
- Headline numbers.

A reviewer with the manifest and your `preds.jsonl` should be able to re-run the grader and get identical verdicts. A reviewer with the manifest and your prompts should be able to re-run predictions and get within-noise numbers.

## Starter guidance

- **Do the sanity check first.** Run the gold-patch evaluation on 20 random instances before generating any model predictions. If gold doesn't resolve at ≥ 95% on those 20, your Docker images are broken and every model number will be dominated by environment failure.
- **The published number is a tuple, not a scalar.** When you pick an anchor, capture the tuple: model + version + harness + version + budget + prompt-set + date. A leaderboard entry from 3 months ago against a since-updated harness is not a comparable anchor — the harness moved.
- **Log cost as you go.** Have the run driver append per-instance dollar cost to a running total; abort if the total exceeds a hard cap you commit to before starting. `$X spent` is not a good post-hoc explanation for why you couldn't finish.
- **Don't chase the last 2 points.** A within-2-points reproduction on Verified is a successful reproduction. Do not spend two more days re-tuning the prompt to close a 1.5-point gap — the noise floor is around that size.
- **Handle patch-format failures explicitly.** A large `% Applied - % Resolved` gap usually means the model produces valid patches that are wrong; a large `100% - % Applied` gap means the model produces malformed patches. The former is a capability signal; the latter is a prompt-engineering fix (usually clearer instructions about diff format, or a post-processor that normalizes headers).
- **Report the per-repo breakdown.** `% Resolved = 42%` is uninformative next to `django: 51%, sympy: 22%, requests: 60%, ...`. The per-repo pattern is what a reviewer uses to sanity-check that your run isn't dominated by one outlier repo.
- **Cite the leaderboard entry date.** SWE-bench Verified's leaderboard changes; your anchor number may be different by the time your reviewer looks. Include the URL and the date.

## Acceptance criteria

- Gold-patch sanity check passes at ≥ 95% resolved on at least 20 instances before model predictions are run.
- Prediction file (`preds.jsonl`) exists for every instance you claim to have run and matches the SWE-bench schema.
- Evaluation report contains `% Resolved`, `% Applied`, `% Environment-error`, and a per-repo breakdown.
- Gap versus the anchor is either within tolerance (≤ 2 points on Verified, ≤ 4 on Lite) or attributed with evidence to one of the four Chapter 3 sources.
- The manifest pins model, harness, dataset revision, and Docker image digests such that a reviewer can re-run the grader and reproduce identical verdicts.
- If your harness emits traces, Part E reports the resolved-vs-failed cost ratio.

## Stretch goals

- **`pass^k` reporting.** Run 3–5 epochs against the same instances at temperature > 0 and report `pass^k` (all `k` epochs pass) alongside `pass@1`. Include the fraction of instances that are "unstable success" (pass on some epochs, not others). Note the extra cost in the manifest.
- **Harness A/B.** Run the same model against two harnesses (e.g. SWE-agent and Aider). Report the gap and attribute to prompts vs tools vs error-handling. This is the strongest evidence that "SWE-bench score" is not a property of a model alone.
- **Cost-vs-accuracy Pareto plot.** Plot `mean $ per instance` vs `% Resolved` for (a) different budgets on the same model + harness, (b) different models on the same harness. Report the Pareto frontier as the interesting result.
- **Environment-error investigation.** Pick 5 instances that returned `test_errored`, dig into the container logs, and file a one-paragraph diagnosis per instance. Some will be genuine model bugs; some will be Docker resource issues; some will be dataset-quality issues. The diagnosis process is a durable skill.
- **Adapter into the exercise-01 canonical schema.** Write a converter from SWE-agent trajectory JSON (or your chosen harness's format) into the canonical schema, and score with the exercise-01 aggregator. Compare the aggregator's `no_answer` rate to `1 - % Applied`; they should match up to definitional differences (a "no submit" trajectory versus a malformed patch that didn't apply).
