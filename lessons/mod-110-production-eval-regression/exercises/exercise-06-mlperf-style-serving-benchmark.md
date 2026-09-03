# exercise-06: MLPerf-Style Serving Benchmark

**Estimated effort:** 2 hours

## Objective

Build an internal serving benchmark for one LLM inference stack that follows the MLPerf Inference methodology from Chapter 7. The benchmark reports the four numbers (TTFT, TPOT, throughput, accuracy against a floor) across at least two of the four MLPerf scenarios (Offline and Server at minimum), on a stratified workload derived from your team's actual request shape, with a reproducibility manifest complete enough that another team could re-run it and compare. The deliverable is not a single number — it is a report structured so that infrastructure decisions (which GPU, which inference engine, which quantization tier) can be defended.

## Prerequisites

- mod-110 Chapter 7 (MLPerf-style serving benchmarks), Chapter 6 (observability integration, for wiring the benchmark result into the release process).
- MLPerf Inference reference (Reddi et al. 2020) — the methodology. LoadGen documentation for the request-generator harness.
- Familiarity with at least one modern LLM inference engine: `vLLM`, `SGLang`, `TensorRT-LLM`, or `Text Generation Inference (TGI)`.
- Access to at least one GPU capable of running the target model — cloud (e.g., an on-demand H100 hour), workstation (A100 / L40S / consumer 3090 / 4090), or a small enough model that CPU-only Metal / AVX inference is viable for the exercise (a 1–3B model on `llama.cpp` still lets you exercise the methodology, if not the production-scale numbers).
- Python 3.11+ with `numpy`, `pandas`, plus the inference engine's client library.

## Workload

Author `workloads/chat.jsonl` — a pinned request corpus that stratifies your team's real request shape (or a public analogue if you have no production access). Each row:

```json
{"id": "req_00001", "stratum": "chat-medium", "input_tokens": 517, "output_tokens_target": 203, "prompt_text": "..."}
```

Suggested strata (adapt to your product):

- `chat-short` (input ≤ 200 tokens, output ≤ 100).
- `chat-medium` (input 200–1500 tokens, output 100–500).
- `chat-long-context` (input 5k–20k tokens, output 100–500).
- `code-generation` (input 200–1000 tokens, output 500–2000, higher variance).
- `tool-call` (input 500–2000 tokens, output ≤ 100 tokens as a structured JSON reply).

Pin the corpus by SHA. Report per-stratum results, never aggregate across strata for a headline number — different strata have different SLA shapes.

## Model and reference accuracy set

- Pick one target model: e.g., `Llama-3.1-8B-Instruct`, `Llama-3.1-70B-Instruct`, `Mistral-7B-Instruct`, or `Qwen-2.5-7B-Instruct`. Pin by HuggingFace revision SHA.
- Author or reuse a **reference accuracy set** (e.g., 200 items from `gsm8k` `main`, or 200 items from an internal correctness benchmark) with gold answers. This is the accuracy floor gate: the benchmark's output is run through the reference set at the end of every scenario run, and the result is invalid if accuracy falls below the pre-committed floor (e.g., `≥ 99% of the FP16 reference accuracy` or `≥ 99.9%`, per MLPerf's two named tiers).

## Requirements

### Part A — the benchmark charter

Ship `BENCHMARK_CHARTER.md` *before* running the benchmark. It commits to:

- The target model and revision SHA.
- The inference stack (engine, version, containerized configuration).
- The hardware (GPU model, count, firmware, host CPU, memory, PCIe topology).
- The four SLA thresholds per stratum: TTFT P95 (ms), TPOT mean (ms), total latency P95 (ms), throughput target (req/s or tok/s).
- The accuracy floor and the reference accuracy set (SHA-pinned).
- The scenarios that will run and their configurations.
- The warm-up policy (N requests or M seconds before measurement begins).
- The number of repeated runs per (scenario, stratum) and the accepted run-to-run variance.

Nothing in the benchmark result is defensible without the charter; the CI script should refuse to publish a report if the charter is missing.

### Part B — the harness

Ship `bench/harness.py` — a request driver that supports the two required scenarios and returns per-request timing:

- **Offline scenario.** Submit all requests concurrently (bounded by an `--max-concurrency` flag, typically the inference engine's max batch size). Measure end-to-end throughput (tokens/second and requests/second) and accuracy on the reference set.
- **Server scenario.** Submit requests according to a Poisson-distributed arrival pattern at a specified target rate. Measure TTFT, TPOT, total latency (per-request), and count SLA violations. If the SLA-violation rate exceeds a pre-committed threshold (typically 5%), the run *fails* — the target rate was too high for the SLA.

Both scenarios record per-request: `request_id, arrival_time, first_token_time, last_token_time, input_tokens, output_tokens, error`.

### Part C — accuracy checking

Ship `bench/accuracy.py`:

- For each scenario run, feed the reference accuracy set through the same serving stack (not a separate model) and grade the outputs against gold.
- Report accuracy as a fraction and as a fraction of the FP16 reference accuracy.
- Fail the run if accuracy falls below the pre-committed floor.

The accuracy check must run under the same stack configuration as the benchmark run — quantization, batching, speculative decoding, etc., all in effect. Chapter 7's warning: quoting throughput at INT4 and accuracy at FP16 is the standard misdirection to design against.

### Part D — the reproducibility manifest

Ship `bench/manifest.py` that captures:

- Hardware: GPU model + count + firmware, host CPU, memory, PCIe topology, NIC.
- Software: OS, kernel, CUDA / driver version, PyTorch / vLLM / TensorRT-LLM / SGLang / TGI version, tokenizer package version.
- Weights: HF revision SHA, quantization tier (`fp16`, `bf16`, `int8`, `fp8`, `int4`), quantization tool version.
- Workload: `workloads/chat.jsonl` SHA, per-stratum sample counts.
- Warm-up: policy, count, duration.
- Runs: count per (scenario, stratum), seeds.
- Dates and operator.

The manifest is emitted as `manifest.json` alongside the report and referenced by SHA in the report's header. Two independent runs of the same manifest on the same hardware must be reproducible to within the reported run-to-run variance.

### Part E — the report

`python -m bench.report results/ --charter BENCHMARK_CHARTER.md --manifest manifest.json --out report.md` produces:

- Per (stack, hardware, model) triple, one row per (scenario, stratum) with:
  - TTFT P50 / P90 / P95.
  - TPOT mean and P95.
  - Throughput (tokens/s and req/s).
  - Accuracy vs. floor (PASS / FAIL).
  - Run count and spread (min–max, IQR) across runs.
  - Verdict against the charter's SLA and accuracy floor.
- A "reading list" section that names anything that varied between runs beyond the manifest (should be empty — variation is a bug).
- The manifest excerpt at the head so the reader can see the environment without opening a separate file.

The report shape follows Chapter 7's example row for the Server scenario, chat workload.

### Part F — the launch-review wiring

Add a `regsuite`-style gate (from exercise-01) that consumes the benchmark's `verdict.json` and blocks a candidate release when the Server-scenario TTFT P95 or the accuracy floor regresses versus the incumbent stack. This is Chapter 7's "wire the benchmark into the release process" — the benchmark is not a one-off characterization; it is a release artifact whose regression blocks a ship.

## Starter guidance

- **The charter comes first.** Every mistake in this exercise traces to running the benchmark and then figuring out what numbers to report. Author `BENCHMARK_CHARTER.md`, commit it, then write the harness.
- **Pick the right scenario for the product.** Offline throughput is not the interactive-chat number. If your target is a chat product, the Server scenario at the SLA-required arrival rate is the answer. Chapter 7's confusion #1.
- **Warm-up is not optional.** LLM stacks have cold-start penalties (KV cache initialization, JIT compile, weight loading). If you measure without warm-up, your first-run TTFT is 3–10× larger than steady-state, and the reported number is misleading.
- **Repetition, not single-shot.** A single benchmark run has 5–15% run-to-run variance. Three runs and reporting the median + spread is the minimum; five is better.
- **Accuracy on the same stack.** Quantization, speculative decoding, and aggressive batching can each cost 0.5–2% accuracy silently. The accuracy check must run under the exact stack configuration that produced the throughput number.
- **The manifest is not documentation; it is the contract.** Another team cannot reproduce your number without knowing the CUDA version and the driver. If your manifest omits a field a reader would need, they cannot act on the number.
- **MLPerf compliance is not required for the exercise.** The methodology is. Full MLPerf submission requires the LoadGen harness, the exact reference workloads, and a peer-review process. This exercise adopts MLPerf's discipline; a submission is a stretch goal.

## Acceptance criteria

- `BENCHMARK_CHARTER.md` exists and was committed before the harness code (verifiable via `git log`).
- The harness runs both Offline and Server scenarios end-to-end on at least one stratum and produces per-request timing records.
- The accuracy check runs under the same stack configuration as each scenario run; a run whose accuracy falls below the charter's floor is reported as FAIL.
- The reproducibility manifest is emitted per run and includes hardware, software, weights, workload, warm-up, and dates.
- The report includes at least two strata, both scenarios, at least three runs per (scenario, stratum), and reports the run-to-run spread.
- A synthetic regression injection (deliberately quantize the model to a lower tier that fails the accuracy floor) is caught by the report as FAIL and by the launch-review gate as a block.
- The report contains no aggregate-across-strata headline number that would misrepresent a specific product surface's SLA.
- Two independent runs of the same manifest reproduce the same headline numbers within the reported variance envelope.

## Stretch goals

- **Multi-scenario comparison.** Add Single-Stream (batch size 1, back-to-back) for the edge / single-user reference; some SDKs and single-request user experiences are best characterized at this scenario. Report all three scenarios on the same table so the axis trade-offs are visible.
- **Stack sweep.** Run the same charter against two engines (e.g., `vLLM` vs. `SGLang`) on the same hardware and same model. Reconcile the differences — expect 10–30% throughput divergence — and document the winning engine per stratum. This is what the benchmark exists to enable.
- **Quantization tier sweep.** For a fixed engine and hardware, sweep quantization tiers (FP16 → BF16 → FP8 → INT4) and plot throughput vs. accuracy. Chapter 7's accuracy-floor discipline picks the winning tier at the highest throughput that still passes the floor.
- **Real Poisson arrivals.** Replace the harness's default Poisson generator with a trace-replay from real production request timestamps (with PII scrubbed). Compare Server-scenario numbers under synthetic Poisson vs. real-arrival trace; expect the tail to widen under real arrivals because production traffic is not Poisson.
- **Wire into the observability platform.** Emit each benchmark run's per-request timing as spans in Phoenix / Langfuse / Weave (from exercise-05). The benchmark becomes queryable alongside production traces — same schema, same UI.
- **Full MLPerf compliance run.** Adopt the MLCommons LoadGen harness, run one of the MLPerf reference workloads (Llama-2-70B or Llama-3.1-405B, depending on your hardware), and produce a submission-shaped result package. Not the goal of the exercise, but the natural extension for a team preparing an actual MLPerf submission.
