# exercise-01: pass@k with Sandbox Execution

**Estimated effort:** 3 hours

## Objective

Build a functional-correctness eval end-to-end for a code-generation benchmark: a sandboxed executor with a proper timeout and resource-limit policy, the Chen et al. 2021 unbiased pass@k estimator, and a runnable pipeline over HumanEval and HumanEval+ that reports pass@1 and pass@10 for one model. The deliverable is the *scorer*, not a model — every configuration knob, from sandbox to decoding, is exposed and defensible.

## Prerequisites

- mod-107 Chapter 2 (pass@k and the sandboxed execution loop).
- mod-104 Chapter 7 (prompt-format sensitivity and reproducibility).
- A machine (or container host) where you can run untrusted Python in isolation. Docker is strongly recommended; if you cannot use Docker, `subprocess` + `resource.setrlimit` is a viable fallback for HumanEval-scale problems only.
- Access to at least one code-generation model. Any of: a hosted API (OpenAI, Anthropic, DeepInfra), a local Llama-3-Instruct, or a small dedicated code model (StarCoder2, Qwen2.5-Coder, DeepSeek-Coder) served via vLLM or Ollama.

## The benchmark

Use **HumanEval** (Chen et al. 2021) as the primary benchmark and **HumanEval+** (Liu et al. 2023 / EvalPlus) as the augmented-test comparison. Both are ~164 problems; the difference is that HumanEval+ expands the per-problem test count by roughly 80×. Do not use MBPP or LiveCodeBench for this exercise — HumanEval is small enough to iterate on quickly and well-understood enough that reference numbers exist for cross-check.

Do not use SWE-bench-scale benchmarks either; the sandbox complexity is a different exercise (touched on in mod-108).

## Requirements

### Part A — the sandbox executor

Ship `sandbox/executor.py` exposing:

```python
def run_sample(problem: dict, sample_code: str, timeout_s: float) -> ExecutionResult:
    """Execute `sample_code` against `problem["test"]` under sandbox. Return
    ExecutionResult(status, stdout, stderr, elapsed_s, exit_code).

    Status enum: passed, failed, timeout, exception, mem_exceeded.
    """
```

The sandbox must enforce, at minimum:

1. **Process isolation.** Each sample runs in its own subprocess or container. A sample's crash cannot poison sibling runs. Use `subprocess.run` with `timeout=` (never `signal.alarm`, which is defeated by signal-catching samples); prefer a fresh container per sample if using Docker.
2. **Wall-clock timeout.** Configurable per-call; default 5 seconds. A sample that exceeds the timeout is `status=timeout` (not `failed`). The timeout must be a hard kill (`SIGKILL` after `SIGTERM`).
3. **Memory cap.** 1 GB default. Use `resource.setrlimit(RLIMIT_AS, ...)` in a `preexec_fn` or the container equivalent (`--memory=1g`). A sample that trips it is `status=mem_exceeded`, not `failed`.
4. **Filesystem isolation.** Run in an ephemeral tempdir CWD. No writes outside the tempdir. If containerized, mount the tempdir read-write and nothing else.
5. **Network isolation.** No outbound network. `--network=none` in Docker, or verify absence of network egress in the subprocess variant (block via iptables / user namespace if the host allows it).

Log every sample's status, stdout, stderr, and elapsed time. The executor is the *load-bearing* part of this exercise; ship a `test_executor.py` (see Part D) that probes it.

### Part B — the pass@k estimator

Ship `estimator.py` with:

```python
def pass_at_k(n: int, c: int, k: int) -> float:
    """Chen et al. 2021 §2. n samples per problem, c passing, report pass@k."""
```

Implement the closed form:

```
pass@k = 1 - C(n - c, k) / C(n, k)   if n - c >= k
       = 1                            otherwise
```

Do **not** use `float(k*c) / n`; the naive fraction is biased for `k < n`. Ship a unit test that exercises the boundary cases (`c = 0`, `c = n`, `n - c < k`, `n = k`, and the numerical stability at large `n`).

### Part C — the run driver

Ship `run.py` that:

1. Loads HumanEval and HumanEval+ from their reference distributions (via `evalplus` or `datasets`).
2. For each problem, samples `n = 10` completions from the model at temperature 0.6 with `top_p = 0.95` (justify any deviation in the report).
3. For each completion, runs the sample through the sandbox against the problem's test suite.
4. Records per-sample status (as an ExecutionResult) in `logs/samples.jsonl`.
5. Aggregates per-problem `c` (passing count) and computes pass@1 and pass@10 via the estimator, on both HumanEval and HumanEval+ test suites.
6. Reports timeout rate and mem-exceeded rate as first-class metrics next to pass@k.

The driver must be idempotent: re-running against an existing `logs/samples.jsonl` should skip completed samples and only run remaining ones.

### Part D — the scorer probe suite

Ship `test_executor.py` that runs the sandbox against a handful of seeded samples and asserts the scorer's behavior. At minimum:

- **Known-correct.** Feed HumanEval's canonical reference solutions from the dataset. Expected: `status=passed` for every problem.
- **Known-wrong.** A trivial `return None` per problem. Expected: `status=failed` (assertion error or wrong output), not `exception`, not `timeout`.
- **Infinite loop.** `while True: pass`. Expected: `status=timeout` within 5.5s (small margin over the 5.0s cap); the process is terminated, not left running.
- **Memory hog.** `x = [0] * (10**9)`. Expected: `status=mem_exceeded`; must not OOM the host.
- **Syntax error.** `def foo(: return 1`. Expected: `status=exception`.
- **Sandbox escape attempt.** A sample that writes to `/tmp/escape.txt` outside the ephemeral tempdir. Expected: either write is scoped to the tempdir, or the write fails. The test asserts that no artefact is created outside.
- **Network attempt.** A sample that `import urllib.request; urllib.request.urlopen("http://example.com")`. Expected: exception or timeout, not success.

A scorer that does not pass every one of these probes is not a scorer whose numbers you should publish.

### Part E — the report

Ship `REPORT.md` (≤ 2 pages) covering:

1. **Setup.** Model, provider, decoding params, sandbox implementation (subprocess vs Docker), and the exact benchmark version hashes.
2. **Numbers.** pass@1 and pass@10 on HumanEval and HumanEval+; timeout rate; mem-exceeded rate; parse-failure rate.
3. **The HumanEval → HumanEval+ gap.** By how many points does pass@1 drop when you switch to the augmented tests? Which problem categories drop most? (This is a direct measurement of your model's over-fit to the original tests.)
4. **Scorer probe results.** All eight probes from Part D with pass/fail status.
5. **Reproducibility manifest.** Model version, dataset commit / package version, harness version, seeds, and the `run.py` invocation.

### Part F — bundle

- `sandbox/executor.py`, `sandbox/Dockerfile` (if containerized)
- `estimator.py`, `test_estimator.py`
- `test_executor.py`
- `run.py`
- `logs/samples.jsonl` (partial log is fine; do not commit huge logs — a sample-of-samples for reproducibility is enough)
- `REPORT.md`
- `README.md` — one-shot rerun instructions

## Starter guidance

- **Start with the probe suite, not the benchmark.** Get all eight scorer probes passing on a stub `run_sample` before you run a single model sample. If the sandbox mis-handles the infinite loop probe, the HumanEval numbers are noise.
- **Do not use `exec()` in-process.** Every mature code-eval harness runs samples in a subprocess or container. In-process `exec` shares memory, imports, and the ability to `sys.exit(0)` out of your eval driver. Do not do it.
- **Default to Docker.** The subprocess + `resource.setrlimit` route works for HumanEval scale but is fragile — signal-catching samples, subprocess-spawning samples, and any sample that forks all defeat single-process resource limits. If your target environment has Docker, use it. Ship a `Dockerfile` and a `docker run` invocation.
- **Truncate at stop sequences.** HumanEval prompts end with a function signature; the sample is a *body*. Set stop sequences to `["\nclass ", "\ndef ", "\n#", "\nif __name__"]` (or whatever your model uses for top-level definitions) so the sample is only the completion body. Otherwise the model continues writing new top-level definitions and the tests fail for the wrong reason.
- **Compare against a published number.** Any model with a public HumanEval pass@1 is a comparability anchor. If your number is > 5 points off the public number for the same model, one of: (a) the decoding params differ, (b) the sandbox is timing out samples that should pass, (c) the truncation is wrong. Diagnose before shipping the report.
- **Report timeout rate.** A HumanEval pass@1 of 0.55 with timeout rate 0.02 is fine. A HumanEval pass@1 of 0.55 with timeout rate 0.18 is a red flag — the model is producing hangs and the "wrong" bucket has quietly absorbed them. This is the number most implementations forget to publish.
- **The `n = 10` budget is a choice.** Chen et al. 2021 use `n = 100` for pass@100; the estimator handles any `(n, k)` with `n >= k`. For a 3-hour exercise, `n = 10` is enough to demonstrate pass@1 and pass@10 without burning your token budget on a single benchmark. If you have more time and quota, run `n = 100` and report pass@1 / pass@10 / pass@100 from the same samples.
- **Do not use the same test suite for training and testing.** You are almost certainly downloading a HumanEval variant that has been used to train the model you're evaluating. Contamination is out of scope for this exercise, but note it in the report; a saturated HumanEval number is not a strong claim.

## Acceptance criteria

- `sandbox/executor.py` passes all eight probes in `test_executor.py`, including the infinite loop, memory hog, sandbox-escape, and network-attempt probes.
- `estimator.py` implements the Chen et al. 2021 closed form correctly, verified against a table of `(n, c, k)` values matching the paper's Table 1.
- `run.py` produces pass@1 and pass@10 on HumanEval and HumanEval+, plus timeout and memory-exceeded rates, in an idempotent driver that can be re-run without redoing completed samples.
- `REPORT.md` includes numbers, the HumanEval → HumanEval+ gap, and the probe pass/fail table.
- The bundle is reproducible: a reviewer with your model access and Docker can rerun `python run.py` and get numbers within decoding-noise of yours.

## Stretch goals

- **`n = 100` sampling, pass@100.** Repeat the run at `n = 100`. Report pass@1, pass@10, pass@100 from the same samples. The estimator is the whole point.
- **Temperature sweep.** Run the same benchmark at temperature ∈ {0.0, 0.2, 0.6, 0.8, 1.0}. Plot pass@1 and pass@10. The pass@1 curve should peak around 0.2; pass@10 should peak higher. This is the empirical support for the "temperature by `k`" heuristic in Chapter 2.
- **MBPP+ as a second benchmark.** Run the same driver on MBPP+ (~500 problems). Report the same three numbers. The difference in per-model rank between HumanEval+ and MBPP+ is diagnostic — models optimized for HumanEval-shape prompts often lose relative ground on MBPP-shape prompts.
- **LiveCodeBench post-cutoff subset.** Filter LiveCodeBench to problems published *after* your model's stated training cutoff. Run the same driver. The gap between HumanEval pass@1 and post-cutoff LiveCodeBench pass@1 is your best available contamination signal without labelling any items yourself.
- **A malicious-sample probe.** Add a sample to `test_executor.py` that attempts to `os.remove` a decoy file the driver placed in `/tmp`. Verify the sandbox prevents it. This is not a real attack surface at HumanEval scale, but it is the discipline that scales to any code-eval production system.
