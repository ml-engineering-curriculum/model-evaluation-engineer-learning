# pass@k and the Sandboxed Execution Loop

Code-generation evaluation was the first place in the modern LLM eval literature where the community accepted that string-match against a reference is the wrong shape for the task. Two syntactically different programs can both be correct. A program byte-identical to the reference can be wrong because the prompt asked for a different signature. The only measurement that carries meaning is *does the program pass the tests you hold out*. That is the entire premise of pass@k, and the reason HumanEval (Chen et al. 2021), MBPP (Austin et al. 2021), and their successors displaced BLEU-vs-reference within a year of the first Codex paper.

The metric itself is short to state. The engineering underneath it — a sandbox to execute untrusted model output safely, a timeout policy that distinguishes wrong from slow, resource caps to keep a runaway sample from taking down the eval host, and the sampling budget that makes pass@k an *unbiased* estimator — is where the exercise lives. Skipping that engineering does not save time; it produces a number that quietly conflates "the model got it wrong" with "the sandbox hung" and "the timeout was too short for the test."

## The metric

For each problem in the benchmark, generate `n` samples from the model (with sampling temperature > 0 so the samples are distinct). Run each sample against the hidden test suite. Let `c` be the number of samples that pass all tests. Then pass@k for that problem is the probability that at least one of *k* samples drawn uniformly *without replacement* from the `n` samples passes. Chen et al. 2021 (§2) give the unbiased estimator:

```
pass@k = 1 − C(n − c, k) / C(n, k)      when n − c ≥ k
       = 1                              when n − c < k
```

where `C(a, b)` is the binomial coefficient. The benchmark-level pass@k is the mean of that quantity across problems.

Two implementation traps here that show up in almost every homegrown eval:

- **Do not compute `k / n` from a `n = k` run.** The naive fraction `c / n` is biased for `k < n`; the estimator above is the correction. If you evaluate at `n = 1` and report pass@1, `c / n` and the estimator agree; the moment you evaluate at `n = 100` and want to report pass@1 *and* pass@10 *and* pass@100 from the same samples, the estimator is the only correct choice.
- **Sample `n` once; report multiple `k` from the same `n`.** Sampling `n = 100` and computing pass@1, pass@10, pass@100 from that one run is cheaper and lower-variance than three separate sampling runs. The estimator makes this correct.

The `k` values reported in the Codex paper are pass@1, pass@10, and pass@100. Most subsequent papers report at least pass@1; higher `k` is diagnostic ("the model can do this if it gets several tries") but a poor proxy for a production system that will only get one try per user turn.

## The sandbox is not optional

Model-generated code is untrusted input. The Codex paper (Chen et al. 2021, §7 "Hazard analysis") is explicit about this: samples may contain infinite loops, filesystem writes, network calls, forked processes, or `os.system("rm -rf ~")`. The HumanEval reference implementation *disables* execution by default and requires the operator to enable it explicitly with a code change, precisely because a naive `exec(sample)` on an eval host is a security incident.

A sandbox for code eval has to enforce, at minimum, five properties:

1. **Process isolation.** Each sample runs in its own subprocess (or container). One sample's crash cannot poison another sample's environment; one sample's `sys.exit(1)` cannot end the eval.
2. **Wall-clock timeout.** A per-sample timeout kills runaway samples. Typical bounds: 3–10 seconds for HumanEval-scale problems, longer (30–120 s) for BigCodeBench / SWE-bench-scale problems. The timeout must be a *hard* kill — a `signal.alarm` inside the sample process is defeated by any sample that catches `SIGALRM`, and `threading.Timer` cannot preempt CPython bytecode. Use `subprocess` with `timeout=` and reap on TimeoutExpired.
3. **Memory cap.** `resource.setrlimit(RLIMIT_AS, ...)` (or a container memory limit) prevents a sample that allocates a giant list from OOM-ing the host. 1–2 GB is usually enough headroom.
4. **Filesystem isolation.** Either an ephemeral temp directory as CWD with no write access outside it, or a full container. `os.chdir(tempdir); os.chroot(tempdir)` is the minimum; a Docker container or gVisor sandbox is the correct answer for anything running unattended on shared infrastructure.
5. **Network isolation.** No outbound network. A sample that curls a beacon out on failure is a data exfiltration vector; a sample that makes real API calls introduces flakiness and cost. Enforce with a container network policy (`--network=none`) or a firewall rule; do not rely on the model "not doing that."

Every mature code-eval harness ships this. The BigCode `bigcode-evaluation-harness` executes samples inside Docker with per-container timeouts and disabled networking. lm-evaluation-harness's code tasks similarly gate execution behind an environment flag and use subprocess isolation. Inspect (UK AISI) has first-class `sandbox` primitives (Docker or local) exposed in every scorer that runs generated code. HumanEval's canonical `execution.py` uses subprocess isolation with a wall-clock guard and disables network via a `guard.py` monkeypatch. If you are rolling your own executor, start from one of these; do not reinvent the resource-limit layer.

## Timeouts are semantic, not operational

The timeout is not a performance parameter. It is part of the *definition of correctness for this benchmark*. A test that times out is scored as failed. That means the timeout budget silently answers a question the benchmark author had to decide: *do we count a solution that produces the correct output in exponential time as a solution?* Most code benchmarks say no — a HumanEval problem that solves `two_sum` by scanning all pairs is not "wrong" but also cannot be allowed to hang the eval for a minute.

Two failure modes here:

- **Timeout too tight.** The reference solution passes the tests, but your sandbox timeout kills the model's solution on a legitimately slow input. Any benchmark you did not personally profile can bite here — HumanEval's harder problems and BigCodeBench's data-science tasks include tests that take multiple seconds on a cold Python interpreter. Set the per-sample timeout by profiling the *reference* solution on the test suite (3–5× reference time is a defensible default) and log timeout-vs-fail separately.
- **Timeout too loose.** A model sample with an infinite loop passes into a 5-minute wall until you kill the job. Every `n = 100` run has a handful of these. The right posture is a hard timeout in the low seconds *plus* logging the timeout fraction as a first-class metric — high timeout rates suggest the model is producing plausible-looking but non-terminating loops.

Report timeout rate next to pass@k. A number like "pass@1 = 0.42, timeout rate = 0.11" is meaningfully different from "pass@1 = 0.42, timeout rate = 0.002," and the two-metric report is nearly free once you already log per-sample outcomes.

## Sampling temperature and the estimator

pass@k with `k > 1` presumes distinct samples. Greedy decoding (temperature = 0) produces the same sample every time; pass@100 at temperature 0 equals pass@1. Chen et al. 2021 sample HumanEval at temperature 0.2 for pass@1 (where the metric is close to accuracy under greedy) and temperature 0.8 for pass@10 and pass@100 (where diversity matters). Modern reports vary — some use temperature 0.6 uniformly; some use nucleus sampling — but the principle is consistent: at higher `k`, use a higher temperature, or the estimator is scoring an under-diverse sample.

Two hygiene points that trip up first-time implementers:

- **Log the temperature next to pass@k.** "pass@10 = 0.61" without a temperature is uninterpretable; the same model, same problems, at temperature 0.2 vs 0.8, can differ by 5–10 points at high `k`. Every published pass@k number should carry a decoding-parameter tuple.
- **Fix a seed per (problem, sample-index).** Reproducibility of a pass@k run means someone else can rerun with the same model, same decoding params, same seeds and get the same per-sample outcomes. Some providers do not honour seeds strictly; for those, log the number of samples and treat the number as an aggregate rather than a per-sample-reproducible artefact.

## Test-suite integrity

Every pass@k number is only as meaningful as the test suite behind it. The single largest quiet finding in code-eval methodology is that many benchmarks under-test — an original HumanEval problem might have 3–8 tests, so a sample that passes those tests may fail on a slightly different input. The community responded with augmented test suites:

- **HumanEval+** (Liu et al. 2023, *Is Your Code Generated by ChatGPT Really Correct?*) expands HumanEval's per-problem test count by roughly 80× via automated test generation. Reported pass@1 drops on many models by 10+ points when tests are expanded, which is a direct measurement of how much the original tests over-report.
- **MBPP+** applies the same treatment to MBPP.

The eval-engineering takeaway is that "we ran HumanEval" and "we ran HumanEval+" are different measurements and cannot be compared. When you cite a pass@k number, cite the test-suite version, not just the benchmark name.

Two other test-suite pitfalls:

- **Contamination.** HumanEval, MBPP, and their + variants are widely reproduced online and are almost certainly in most pretraining corpora. Contamination is a mod-102 topic; the practical implication here is that a headline HumanEval pass@1 is *not* a good proxy for out-of-distribution code generation. LiveCodeBench (Jain et al. 2024) rotates in fresh problems monthly to combat this; it is a better instrument for measuring generalization even though pass@k as a metric is the same.
- **Prompt format.** HumanEval provides a docstring + signature; MBPP provides a natural-language description + a few tests. A "clean" HumanEval prompt versus a `system=You are a Python expert. Complete the following function.` chat wrapper will produce different numbers on the same model. Chat-template sensitivity (mod-104 Chapter 7) applies here too.

## The scorer is the whole eval

Restate the four objects for a code task:

- **Model adapter.** Generation, sampling temperature and top-p, stop tokens (a common bug: models keep generating past the closing `def` if the stop sequence is not `["\nclass ", "\ndef ", "\nif __name__"]` or similar; HumanEval samples truncated correctly are much cleaner).
- **Task definition.** The problem statement, function signature, and the hidden test suite. The eval-engineer owns the test suite; the model author does not see it.
- **Request type.** Generation.
- **Scorer.** The subprocess executor, timeout policy, per-sample pass/fail record, and the unbiased pass@k aggregator. This is the load-bearing part of the eval. Every other choice — prompt format, decoding, model — is a knob turning inputs to this scorer.

If the scorer is the whole eval, then the scorer needs its own test. A common technique: seed the sample set with known-correct reference solutions (should get pass@1 = 1.0 across all problems); seed with known-broken solutions (should get pass@1 = 0.0); seed with an infinite loop (should be counted as failure, not as sandbox hang); seed with a memory-hog (should be caught by the RLIMIT). A scorer that does not pass these seeded probes is a scorer whose numbers you should not trust.

## Beyond HumanEval: the current landscape

HumanEval remains the reference for a reason — it is small, well-understood, and comparable across a decade of papers — but no serious eval suite uses only HumanEval anymore. A short tour of what a code-generation suite actually contains today:

- **MBPP / MBPP+.** Broader coverage than HumanEval; more natural-language-driven prompts; different distribution of task difficulty.
- **HumanEval+ / MBPP+.** Same problems, expanded tests. If you cite HumanEval, cite HumanEval+ next to it; the pair reveals the over-fit-to-tests gap.
- **BigCodeBench (Zhuo et al. 2024).** Practical function-completion tasks with library calls and complex signatures; substantially harder than HumanEval.
- **LiveCodeBench (Jain et al. 2024).** Rolling collection of competitive-programming problems with publication-date-based contamination cutoffs. Useful specifically because you can filter to problems released *after* the model's cutoff.
- **SWE-bench (Jimenez et al. 2024) and SWE-bench Verified (OpenAI/Princeton 2024).** Repository-scale bug-fix tasks with real GitHub issues and test suites. Not a function-completion benchmark; a *software-engineering* benchmark. The scorer is still pass-on-test-suite, but the sandbox now has to run a whole project's tests, and the timeout and resource budgets go up by 2–3 orders of magnitude. Exercise 01 stays on the HumanEval end of the spectrum; SWE-bench-scale eval is called out in mod-108 (agent-and-tool eval).

The pass@k metric is the same across all of these. The sandbox complexity is not: HumanEval fits in a subprocess with a 10-second timeout; SWE-bench needs a per-repo container with the project's install steps run once and cached, plus multi-minute test-suite timeouts.

## Summary

pass@k is a functional-correctness metric — the fraction of problems where at least one of `k` samples passes a hidden test suite, computed from `n ≥ k` samples via the unbiased estimator from Chen et al. 2021. Its four load-bearing implementation details are the sandbox (process, filesystem, network, and resource isolation for untrusted model output), the timeout policy (per-sample wall-clock kill that separates *wrong* from *hung*, with the timeout budget set by the reference solution), the sampling regime (temperature calibrated to `k`, per-sample seed for reproducibility), and the test-suite version (HumanEval versus HumanEval+ are different measurements). The scorer is the whole eval; a scorer that has not been probed against known-good, known-bad, infinite-loop, and OOM seed samples produces a number that quietly conflates model failure with sandbox failure. The next chapter turns to math reasoning, where the load-bearing implementation detail moves from execution to *answer extraction*.
