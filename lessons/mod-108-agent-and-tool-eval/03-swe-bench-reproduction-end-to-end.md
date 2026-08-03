# SWE-bench Reproduction End-to-End

SWE-bench (Jimenez et al. 2024, *SWE-bench: Can Language Models Resolve Real-World GitHub Issues?*, ICLR 2024) is the benchmark that made "agent evaluation" a distinct sub-field. It takes real GitHub issues from twelve widely-used Python repositories, pairs each issue with the human-authored PR that resolved it, and asks a model to produce a code patch that — when applied to the repository at the pre-fix commit — makes the hidden test suite pass. It is the largest, most-cited agent benchmark in circulation and it is the benchmark whose *reproduction gap* teaches the most about agent-eval plumbing. This chapter walks the pipeline end-to-end and calls out where the number you compute can differ from the published number, and what to do about each gap.

## What SWE-bench actually measures

For each of ~2,294 issues in the full SWE-bench test split (500 in SWE-bench-Lite, 500 in SWE-bench-Verified):

- **Input to the model:** the natural-language issue text and a snapshot of the repository at the base commit (the commit immediately before the human fix landed).
- **Output from the model:** a unified-diff patch (git format-patch style) that modifies one or more files in the repository.
- **Success criterion:** apply the patch on top of the base commit; run the *fail-to-pass* tests (tests that failed on the base commit and passed on the human-fix commit); optionally verify the *pass-to-pass* tests (tests that were passing before and should still pass after). If all `FAIL_TO_PASS` tests pass and no `PASS_TO_PASS` test regresses, the instance is `resolved`.

The primary metric is **% Resolved** — fraction of instances that pass both test sets. Secondary metrics: **% Applied** (fraction of predictions whose patch even applies cleanly with `patch -p1 --dry-run`), and increasingly the cost / step / token axes from Chapter 2.

The task is deceptively simple to state and hard to run correctly, for one reason: the ground-truth grader is a real test suite in a real repository at a real commit with real dependencies. Every one of those is a source of environment fragility that the harness has to smooth over.

## The three splits and why they differ

- **SWE-bench (full).** 2,294 instances across 12 repos. The dataset the paper reports on. Substantial fraction of instances have historically-known problems: flaky tests, ambiguous specifications, tests that depend on environmental state.
- **SWE-bench Lite.** 300 instances (post-filter), a subset chosen for reproducibility — simpler patches, single-file fixes, no obvious ambiguity. Almost every recent paper reports on Lite because the full-split noise floor is high enough to obscure model comparisons.
- **SWE-bench Verified.** 500 human-verified instances (OpenAI / Princeton 2024). A team of professional software engineers manually reviewed 1,699 instances and kept only those where the specification is clear, the test suite is deterministic, and the reference PR is a reasonable interpretation. Published as a fresh benchmark with its own leaderboard on August 13, 2024. If you have to pick one split for a serious eval, use Verified — it is the version most aligned with the original paper's intent, filtered for the reproducibility problems the full split has.

Cite the split every time. A "SWE-bench score of 21%" is uninterpretable without the split; the same model can score materially differently on full vs Lite vs Verified.

## The reproduction pipeline in five stages

The SWE-bench harness (`swebench` Python package, upstream at `github.com/princeton-nlp/SWE-bench`) breaks the pipeline into five stages. Every stage has a version-pinning concern.

### Stage 1 — Load the benchmark

```python
from datasets import load_dataset
ds = load_dataset("princeton-nlp/SWE-bench_Verified", split="test")
```

The dataset ships as a Hugging Face dataset. Each row has: `instance_id` (e.g. `django__django-11133`), `repo`, `base_commit`, `problem_statement`, `patch` (the ground-truth patch, for evaluation), `test_patch` (the tests that reveal the bug), `FAIL_TO_PASS` (list of test node IDs), `PASS_TO_PASS` (list of test node IDs), `environment_setup_commit` (the commit whose environment specification the harness will use).

Pin the dataset revision. `load_dataset("princeton-nlp/SWE-bench_Verified", revision="<commit-sha>")` if you need bit-for-bit reproducibility months later. The Verified split has been re-released once already to fix labelling issues; a bare load call will drift silently.

### Stage 2 — Generate predictions

For every instance, run your model against the issue and produce a patch. This is the *agent* stage — the harness is agnostic to how you produce the patch. Three families of approach:

- **Direct patch generation.** The model reads the issue plus a bounded context (retrieved files, or a BM25/embedding shortlist of relevant files) and emits a diff in one shot. Simple but weak on any issue where the model cannot see the right files up front.
- **Agent-style loops.** An agent harness gives the model tools — `read_file`, `write_file`, `edit_file`, `run_tests`, `search_code`, `bash` — and lets it iterate. This is what most recent published SWE-bench numbers use (SWE-agent, OpenHands, Aider, Cursor Agent, Devin, Cognition's harness, and every vendor-supplied agent). The tools, their descriptions, and the harness's error-handling policy all materially affect the number.
- **Retrieval-augmented one-shot.** A hybrid: use retrieval to select files, then one-shot generate the patch conditioned on those files. Cheaper than agent loops, weaker on issues where the right file isn't in the top-k retrieval.

The predictions file is JSONL, one row per instance:

```jsonl
{"instance_id": "django__django-11133", "model_name_or_path": "gpt-4o-2024-08-06", "model_patch": "diff --git a/django/http/response.py..."}
```

`model_patch` is the raw unified-diff text. This is the artefact that gets graded.

### Stage 3 — Build per-instance evaluation environments

For each instance, the grader needs a container with the repository checked out at `base_commit`, the correct Python version, the correct dependency set, and enough of the project's install steps to run the test suite. SWE-bench ships `swebench.harness.docker_build` which:

- Loads a per-instance spec from `MAP_REPO_VERSION_TO_SPECS` — Python version, install command, dependency versions, per-repo shims (e.g. `django` needs `pip install -e .` plus test-runner config; `sympy` needs a numeric-precision-stable random seed).
- Builds a *base image* per (repo, version) tuple.
- Builds an *environment image* per (repo, version, python) tuple layered on the base.
- Builds an *instance image* per instance, checked out at `base_commit` with dependencies installed.

The instance images are cache-heavy — SWE-bench Verified builds ~500 instance images, each ~1–3 GB. First-time build is hours on a workstation and produces ~200+ GB of Docker images. Every re-run reuses the cache; only new instances or changed specs invalidate layers.

For any serious run, do the image build *once* on a machine with a big disk and export the image cache; do not re-build per experiment.

### Stage 4 — Run the evaluation

```bash
python -m swebench.harness.run_evaluation \
  --dataset_name princeton-nlp/SWE-bench_Verified \
  --predictions_path preds.jsonl \
  --max_workers 8 \
  --run_id run_2024_11_15 \
  --report_dir reports/
```

For each prediction, the harness spins up the instance container, applies the patch, runs the `test_patch` to enable the FAIL_TO_PASS / PASS_TO_PASS tests, runs pytest (or the repo-appropriate runner), parses the results, and emits a per-instance verdict:

- `resolved` — all FAIL_TO_PASS passed, no PASS_TO_PASS regressed.
- `applied_but_unresolved` — patch applied cleanly, tests did not all pass.
- `patch_apply_failed` — patch did not apply (syntax, wrong file paths, wrong line numbers).
- `test_errored` — the test runner crashed (usually an environment issue).

Per-instance timeouts default to a few minutes for the test-runner; per-run wall-clock scales with `max_workers`. On modest hardware (~8 CPU, 32 GB RAM), a full Verified run is a few hours after the images are cached.

### Stage 5 — Aggregate and report

The report JSON lists per-instance verdicts. The two headline numbers:

- **% Resolved** = `resolved / total` — the leaderboard number.
- **% Applied** = `(resolved + applied_but_unresolved) / total` — a diagnostic that separates "the model can't produce a valid patch format" from "the model can produce valid patches but they're wrong."

Alongside those, if the prediction file carries usage metadata (some agent harnesses log per-instance token counts and step counts to sidecar files), report the Chapter 2 vector: cost per instance, steps per instance, tool-call validity where available.

## What "reproducing within tolerance" means

If a paper reports `gpt-4o` at `42.8% Resolved` on SWE-bench Verified, and your run of the same harness against `gpt-4o` gives `41.2%`, is that a reproduction? The rough tolerance:

- **±2 points** on Verified (500 instances → SE ≈ 2.2 points at the reported rate, so within-noise). Anything up to ±2 is essentially the same number.
- **±3–4 points** on Lite (300 instances → wider SE).
- **≥5 points** off — investigate before publishing.

The four sources of gap you actually see, in decreasing order of frequency:

### 1. Different agent harness

The published number was produced with a specific harness — SWE-agent, OpenHands, Aider, a vendor's internal harness. The tool prompts, the retry policy, the max-step budget, the format the model is asked to emit patches in, the error-handling scaffold — every one of these moves the number by several points. `gpt-4o` scores materially different on SWE-agent vs OpenHands vs a bare one-shot patch generator; the "gpt-4o SWE-bench score" is not a property of the model alone but of the (model, harness, budget) tuple.

The reproducibility rule: cite the harness and its version alongside the model. `gpt-4o + SWE-agent 0.7.0 + max 40 turns + resolved 42.8%` is a claim; `gpt-4o resolved 42.8%` alone is not.

### 2. Prompt-format sensitivity

The agent harness's system prompt, tool descriptions, and error message templates matter. Small wording changes ("you are an expert software engineer" vs "you are a coding agent") reproducibly move SWE-bench scores by 1–3 points on the same model. This is the mod-104 Chapter 7 phenomenon at agent scale; the fix is the same (pin the prompt, version it, ship it in the reproducibility manifest).

### 3. Sampling variance

SWE-bench is usually run at low temperature (0.0–0.2) with `n=1` per instance — the agent runs one trajectory per issue, and the trajectory itself is stochastic even at temperature 0 (tool-call side effects, timing-dependent test outputs, network flake). Two runs of the same model + harness against the same instances can differ by 1–2 points on Verified from run-to-run stochasticity alone. This is what motivates `pass^k` reporting; if you want a tight number, run 3–5 epochs and report both the mean and `pass^k`.

### 4. Dataset drift or split mismatch

The Verified split has been re-released once; the Lite split has been filtered subtly at least twice. If you loaded the dataset without a `revision=` pin, you may be scoring against a slightly different set from the paper. The tell: your `% Applied` matches or beats the paper but your `% Resolved` is different — the models applied the same patches, but the tests they were graded against differ.

## Non-obvious pitfalls

Six failure modes that are worth naming because they cost every first-time SWE-bench implementer a day each.

### The patch parses but modifies the wrong file

Models emit unified diffs with the file header line `diff --git a/foo/bar.py b/foo/bar.py`. If the model omits the `a/` `b/` prefixes, or uses an absolute path, or picks the wrong package root (`django/http/response.py` vs `/repo/django/http/response.py`), the harness's `git apply` will fail. Predictions with malformed headers all show up as `patch_apply_failed` and count against `% Applied`. Two hygiene points:

- If your prediction pipeline post-processes the model's raw text into a diff, log the raw and post-processed versions per instance. A large gap between "the model produced a valid-looking diff" and "the diff applied" almost always traces to header formatting.
- The SWE-bench harness ships a lenient patch applier that falls back to fuzzy matching for context lines; use it. `git apply --recount --whitespace=fix` recovers a meaningful fraction of patches that would otherwise fail.

### The test suite is flaky

Some SWE-bench instances have tests that pass or fail non-deterministically depending on system time, network availability, or CPU speed. The Verified split was created precisely to filter these out, but a small residual exists. If your pass-to-pass regressions are dominated by a handful of instances, check whether those instances' tests are known-flaky (the SWE-bench issue tracker maintains a list); if so, exclude and note the exclusion.

### The environment install failed silently

Some instances need obscure system dependencies (a specific glibc, a specific C library, a native compiler). If the environment build failed for a particular instance, the harness marks it `test_errored` — but a naive report that computes `resolved / total` counts those as failures against the model. The correct denominator for a model's capability score is instances where the environment built successfully; report `resolved / (resolved + applied_but_unresolved + patch_apply_failed)` alongside the raw number to separate model failure from environment failure.

### The published number is a maximum over configurations

Many published SWE-bench numbers are the best number the team got after tuning agent scaffolding, prompt templates, retry counts, and tool descriptions against a validation split. Reproducing "the paper's number" often means "the paper's best number after tuning on some other subset"; a from-scratch run of the same model may fall a few points short even with the same harness, because you haven't done the tuning. Note this in the report — a within-3-points reproduction on a public benchmark against a paper's tuned setup is a reasonable outcome and not a failed reproduction.

### The base commit isn't what you think

`base_commit` in the dataset is the commit *immediately before* the fix. Some repositories rewrote history after the SWE-bench dataset was constructed; a bare `git checkout <base_commit>` in a fresh clone may fail or resolve to a different tree than the harness expects. The harness handles this with a cached repo mirror pinned to a specific fetch; use it.

### Cost blows up if you naively parallelize

An agent that averages 25 tool calls at ~2K completion tokens per call on `gpt-4o` is ~$3 per instance; a 500-instance Verified run is ~$1,500. If you accidentally set `--epochs 5` or leave a retry loop enabled, that becomes $7,500 quickly. Two habits: (a) run 10 instances end-to-end and extrapolate the bill before starting a full run, and (b) log per-instance cost as the run proceeds and abort early if the running total exceeds a pre-committed cap.

## The reproducibility manifest

Every SWE-bench run report should ship with a manifest that pins:

- **Model.** Provider, model ID, exact version string, temperature, top-p, seed, `max_tokens`.
- **Agent harness.** Framework name and version (`swe-agent==0.7.0`, `openhands==0.9.2`, `inspect-ai==0.3.104`), plus any local patches applied.
- **Prompt.** The system prompt and tool descriptions verbatim (as a file in the run bundle, not summarized).
- **Budget.** `max_turns`, `max_tokens`, `max_wall_clock_s`, `max_dollars` — even if unused, the caps are part of the task.
- **Dataset.** `princeton-nlp/SWE-bench_Verified` at commit `<sha>`.
- **Harness.** `swebench==<version>`, `docker` version, base-image digests for each `(repo, version)` tuple.
- **Environment.** Test runner (`pytest` version), OS, kernel, CPU / RAM headroom.
- **Reporting.** % Resolved, % Applied, per-repo breakdown, cost distribution (median / p95), step distribution, termination-reason distribution, `pass^k` if k > 1.

A reviewer with the manifest and your predictions file should be able to re-run the grading stage and get bit-for-bit the same verdicts. A reviewer with the manifest and your prompts should be able to re-run the prediction stage and get within-noise the same score.

## Where SWE-bench does and doesn't work

**Works well for.** Coarse ranking of "can this model + this scaffolding solve Python engineering tasks." Comparing two models on the same harness. Comparing two harnesses on the same model. Detecting large regressions.

**Does not work well for.** Finely comparing models within 1–2 points on the same harness (noise floor is ~2). Extrapolating to non-Python engineering. Extrapolating to greenfield code writing (SWE-bench is bug-fix / feature-add against an existing codebase; every task's context is a pre-existing repo). Measuring anything about developer experience (developers do many things a SWE-bench task doesn't — code review, incremental refactoring, testing strategy). Measuring "agent-ness" independent of task family — SWE-bench happens to be an agent benchmark, but it is measuring code-repair capability, not agent orchestration.

The eval-engineer's rule: SWE-bench is one instrument. Its results belong in a code-generation section of a product-shaped suite (mod-107 Chapter 7) alongside HumanEval+, MBPP+, and a task-family-appropriate LiveCodeBench slice, not as a standalone claim about a model's engineering capability.

## Summary

SWE-bench is the canonical repository-scale agent benchmark: given a real GitHub issue and the repo at the pre-fix commit, produce a patch that makes the hidden `FAIL_TO_PASS` tests pass without regressing `PASS_TO_PASS`. Reproduction is a five-stage pipeline — load dataset, generate predictions with an agent harness, build per-instance Docker environments, run the grader in containers, aggregate verdicts. The reproduction gap versus a published number is dominated by four sources: different agent harness, prompt-format sensitivity, sampling stochasticity, and dataset-split drift; a within-2-points result on Verified against a paper's tuned setup is a successful reproduction. The published number is a `(model, harness, prompt, budget, dataset)` tuple, not a property of the model alone; every reproducibility manifest must pin the tuple. Chapter 4 turns to the interactive-sandbox benchmarks — WebArena, GAIA, AgentBench — where the environment is a browser or a live filesystem, and the sandbox-isolation and deterministic-replay problems get materially harder than SWE-bench's per-instance containers.
