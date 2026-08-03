# MLPerf-Style Serving Benchmarks: TTFT, TPOT, Throughput Against an Accuracy Floor

The six chapters above measured *what the model produces* — helpfulness, safety, quality deltas under experiments, drift over time. This last chapter measures *how the model serves* — the latency and throughput characteristics of a specific model on a specific stack under a specific workload, holding accuracy above a committed floor. That axis is not a "capability" measurement and not an "experiment" measurement; it is a distinct dimension of the release decision, and it belongs alongside the quality numbers in every launch review.

The reference for this discipline is **MLPerf Inference**, MLCommons' industry-standard benchmark suite (Reddi et al. 2020 for MLPerf Inference; the LLM-specific workloads landed in v3.1 (2023) and have been extended in every subsequent release). MLPerf Inference is not the only serving benchmark, and for many teams a full MLPerf submission is overkill — but its methodology is what any serious internal serving benchmark should structure itself around, and understanding it is prerequisite to reading the vendor numbers that surround GPU procurement, provider selection, and self-hosting decisions.

This chapter walks the four things a serving benchmark measures for LLMs (TTFT, TPOT, throughput, accuracy floor), the four MLPerf scenarios (Offline, Server, Single-Stream, Multi-Stream) and which one matches which product surface, the specific mechanics of running a benchmark whose numbers a stakeholder will actually trust, and the interpretation traps that make one team's "500 tokens per second" incomparable with another team's.

## Four numbers, not one

A single "tokens per second" is almost never the useful reporting shape for LLM serving. The four numbers that together describe a serving configuration:

- **Time to First Token (TTFT).** The wall-clock time from the request arriving at the serving system to the first output token emitted. TTFT is dominated by the prefill phase (the model processing the input tokens) and is what the user perceives as "the model started responding." SLA thresholds are typically expressed at the P90 or P95: "P95 TTFT ≤ 500 ms for chat prompts up to 2000 input tokens."
- **Time Per Output Token (TPOT), also called Inter-Token Latency (ITL).** The average time between subsequent output tokens once generation has started. TPOT is dominated by the decode phase (per-token forward pass through the model). At interactive throughput, TPOT determines the *streaming* speed the user experiences — 30 tokens per second is roughly reading pace, above 60 t/s is faster than a user needs.
- **Throughput.** Two useful sub-numbers here — **request throughput** (requests completed per second, sensible for short-turn workloads) and **token throughput** (input + output tokens processed per second, sensible for aggregate cost accounting). MLPerf's Offline scenario reports the latter as the primary throughput number; the Server scenario reports the request-per-second rate at which SLAs still hold.
- **Accuracy under load.** The model's task accuracy on a reference dataset, measured on the outputs the serving stack produced under the benchmark load. This is the "accuracy floor" — the benchmark result is *invalid* if the model's accuracy on the loaded dataset falls below the pre-committed threshold. Batching, speculative decoding, quantization, and paged-attention tricks all affect accuracy at the margin; the accuracy floor prevents a stack from claiming a huge throughput number that came from a lossy configuration.

The four numbers are related but not substitutable. A stack that has excellent throughput at long batches will have poor TTFT because prefill is starved; a stack tuned for low TTFT will typically have lower throughput because it dedicates capacity to the prefill hot path. A stack with speculative decoding shifts TPOT dramatically at the cost of some throughput and — if drafts are aggressive — some accuracy. The right serving configuration is a Pareto trade-off across the four, not an optimization of any one.

## The MLPerf Inference scenarios

MLPerf Inference defines four scenarios, each modeling a different real-world request pattern. LLM workloads use all four in different submission categories.

- **Offline.** All requests are available at once; the system can batch, reorder, and process them in whatever order maximizes throughput. Metric: total throughput (tokens/second or samples/second). Analogue: batch offline scoring, large-scale generation for evaluation or content indexing.
- **Server.** Requests arrive on a Poisson-distributed schedule at a specified target rate; the system must keep TTFT and TPOT below their SLA thresholds. Metric: the maximum sustained arrival rate at which the SLAs are met. Analogue: interactive chat serving with real user arrivals.
- **Single-Stream.** One request at a time, back to back; no concurrency. Metric: P90 latency per request. Analogue: mobile / embedded inference, single-user local deployments.
- **Multi-Stream.** A fixed small number of concurrent streams (e.g., 8 or 16), each issuing back-to-back requests. Metric: P90 latency at the specified stream count. Analogue: multi-camera or multi-tenant edge inference; less common for large LLMs.

For a customer-facing chat product, the Server scenario is the one that matters. For an offline agentic-eval or batch-generation workload, Offline is the right scenario. Mixing them up is a common way to over-claim: quoting Offline throughput as if it were the number the chat product will see is misleading, because the Server scenario's throughput at the SLA-required arrival rate is typically 30–70% of the Offline throughput on the same hardware.

## SLA-anchored latency: the LLM twist

The Server scenario's SLA thresholds are the load-bearing part of an LLM serving benchmark. Different products have different SLA shapes:

- **Interactive chat.** TTFT is critical (users perceive the delay), TPOT is critical (streaming rate must exceed a comfortable reading speed), tail latencies matter (P95 or P99 SLAs are typical).
- **Voice agents / real-time interactive.** TTFT tighter (sub-500 ms is common; some latency-sensitive voice products target sub-200 ms), TPOT very tight (sub-30 ms per token for real-time TTS pipelines), tail latencies critical.
- **Long-form generation / summarization.** TTFT less critical (a few seconds is acceptable), TPOT less critical (the user is not waiting on the streaming rate), throughput and cost more critical.
- **Agentic / tool-heavy workloads.** TTFT critical for individual tool-call turns, but *total wall-clock* to task completion is the top-line SLA, and it depends on the interaction pattern more than any individual latency number. Chapter 4 of this module and mod-108 both discuss this.

The MLPerf Inference LLM tasks (Llama-2-70B, Llama-3.1-405B, Mixtral-8x7B, GPT-J earlier in the suite's history, and updated references over time) each carry pre-committed TTFT and TPOT thresholds designed for interactive chat serving; a submission is invalid if either threshold is exceeded. Internal benchmarks should adopt the same discipline: **the SLA is committed before the benchmark runs, and any result that fails to meet the SLA is a fail, not a data point.**

## The accuracy floor

MLPerf Inference requires every submission to produce outputs whose accuracy on a reference task (a subset of the model's training-eval task, or a specific dataset chosen for the benchmark) is above a threshold — typically 99% or 99.9% of the reference model's FP16 accuracy on the same dataset. This is the *accuracy floor* that keeps throughput numbers honest.

Why it matters: many serving optimizations trade off a small amount of accuracy for a large amount of throughput. Quantization from FP16 to INT8 (or FP8, or FP4) can double throughput while shifting task accuracy by 0.5–2 percentage points. Speculative decoding is designed to be accuracy-neutral in theory but in practice can introduce small drift if the draft model is poorly tuned. Aggressive batching that reorders prompts can affect KV-cache hit rates in ways that shift generation subtly.

Without an accuracy floor, "we got 3× throughput" can mean "we quantized to 4-bit and now the model is worse but nobody measured." With an accuracy floor, the same claim reads: "at the 99% accuracy floor on the reference task, this stack achieves 3× throughput of the FP16 baseline" — a statement whose meaning is unambiguous.

The specific accuracy target is a choice made in the benchmark design. MLPerf's 99% / 99.9% thresholds are two named tiers ("closed" and "high-accuracy" submissions); a well-designed internal benchmark specifies the threshold in the benchmark's charter and does not allow the target to move to accommodate optimizations that would fail it.

## Reproducibility: the details that matter

A serving benchmark whose numbers a stakeholder will trust is reproducible in the following senses:

- **Hardware pinned.** GPU model, GPU count, GPU firmware version, host CPU model, host memory, host NIC, PCIe topology. A benchmark run on 8×H100 SXM is not comparable to 8×H100 PCIe or 8×H200 without noting the differences.
- **Software stack pinned.** CUDA version, driver version, PyTorch / TensorRT / vLLM / SGLang / TGI version, tokenizer version. LLM serving performance moves 10–30% quarter to quarter as inference engines mature; the version pin is not optional.
- **Model weights pinned.** Not "Llama-3.1-70B" but the specific HuggingFace revision SHA of the weights being served.
- **Workload pinned.** The exact request pattern — input token distribution (or a reference distribution from a published corpus), output token distribution, request arrival distribution — reported with enough detail that another team can replay.
- **Warm-up policy.** LLM serving stacks have cold-start penalties (KV cache initialization, JIT compilation, weight loading). Report the warm-up procedure — typically N warm-up requests before measurement begins — and exclude the warm-up window from the reported numbers.
- **Repetition count.** A serving benchmark's variance is often 5–15% run to run. Report multiple runs and their spread; a single-run number is not defensible.

The MLPerf submission process requires all of the above via a strict submission template. Internal benchmarks that follow the same discipline are usable across teams; internal benchmarks that omit these details produce numbers that whoever ran them believes and that nobody else can act on.

## Reading vendor numbers

Vendor serving benchmarks (GPU vendors, cloud providers, inference-engine authors) are typically published in a form that maximizes the impressive number and minimizes the qualifications. Three specific tricks to watch for:

- **Peak throughput without an SLA.** "Achieves 5,000 tokens/second" — but at what TTFT and TPOT? A configuration that runs enormous batch sizes will have huge throughput and terrible TTFT; the number is real, but it is Offline-scenario, not Server-scenario. Reading the fine print for the batch size and the reported latency distribution is the discipline.
- **Numbers on a specific model / prompt distribution.** "Our stack achieves 3× the throughput of vLLM on Llama-2-7B with 128-token inputs and 128-token outputs." That result may reverse on Llama-3.1-70B with 8k-token inputs and 1k-token outputs. Vendor numbers are only comparable across your workload if the workload matches; run your own benchmark on your workload before making infrastructure decisions.
- **FP4 / INT4 quantized numbers next to FP16 accuracy claims.** Throughput is quoted from the quantized configuration; accuracy is quoted from the FP16 configuration; the two do not co-occur in the actual deployment. Look for the accuracy number *on the same configuration that produced the throughput*.

None of these tricks are unique to LLM serving; classical hardware-benchmark literature has been dealing with them for decades. The discipline is the same: **the meaningful comparison is on your workload, your hardware, your SLA, and your accuracy floor.** MLPerf's discipline is designed to force apples-to-apples comparability; internal benchmarks that adopt the same discipline give internally-actionable numbers.

## Wiring the serving benchmark into the release process

The offline regression suite in Chapter 2 included a `quality.latency.p95` gate as a `warn`-level check; the shadow report in Chapter 3 included full serving-metric distributions on production traffic; the A/B in Chapter 4 included TTFT and cost as guardrails. The MLPerf-style benchmark in this chapter sits alongside those, providing the *controlled* measurement that the others build on:

- **Pre-launch.** Before a candidate model is promoted to shadow, the serving benchmark runs on the target hardware with the pre-committed workload and the pre-committed SLA. The benchmark result is a required artifact in the launch review packet, alongside the offline regression report.
- **Regression.** The serving benchmark's numbers become gate thresholds for the offline regression suite: `serving.ttft.p95_at_serving_config_v2 ≤ 500ms`. Every candidate build re-runs the benchmark; a regression that ships without a serving benchmark rerun is a preventable performance incident.
- **Provider selection.** When adding a hosted-model provider or evaluating self-hosting on new hardware, the benchmark is re-run on the new stack and the numbers feed the cost-and-latency comparison. The benchmark's version-pin discipline is what makes those numbers comparable across providers.
- **Drift detection.** Once in production, serving-metric confidence sequences from Chapter 5 monitor for regressions. When they fire, the diagnostic often includes re-running the controlled benchmark to isolate stack changes from workload changes.

## A minimal internal benchmark shape

For a team that does not need to submit to MLPerf but wants MLPerf-quality internal serving numbers, a defensible minimal benchmark:

- **Workload spec.** A published `.jsonl` file of `(input_tokens, output_tokens_target, prompt_text)` triples derived from a stratified sample of production traffic (with PII stripped). Include multiple strata: short chat, long chat, long-context, code generation, tool-call. Each stratum's numbers reported separately.
- **Scenario spec.** Two runs — Offline (all requests submitted at once) and Server (Poisson arrival at a target QPS). Report both; do not pretend Offline throughput applies to the interactive-chat product.
- **SLA thresholds.** Pre-committed TTFT P95, TPOT mean, and total-latency P95 per stratum. Any Server run that violates its SLA is a benchmark fail.
- **Accuracy check.** Run a fixed reference eval (a subset of your internal correctness benchmark) on the outputs the benchmark generated. Report the accuracy number alongside the throughput number; if accuracy falls below the pre-committed floor, the benchmark result is invalid.
- **Reproducibility manifest.** Hardware, software, model SHA, tokenizer, warm-up procedure, run count, dates.
- **Report shape.** Per (stack, hardware, model) triple: one row per scenario per stratum with TTFT P50 / P90 / P95, TPOT mean, throughput, accuracy vs. floor, run count, variance across runs.

Example report row for the Server scenario, chat workload, single hardware/software config:

```
Config: 8×H100-SXM / vLLM 0.6.x / Llama-3.1-70B-Instruct@abc123 / tokenizer v1
Stratum: chat-medium (input P50 500, output P50 200)
Scenario: Server (Poisson arrivals)
Target arrival rate: 12 req/s
SLA: TTFT P95 ≤ 500 ms, TPOT mean ≤ 20 ms
Runs: 5 (excluding 60 s warm-up each)

Measured (median across 5 runs):
  Sustained rate at SLA:    11.8 req/s  (spread across runs: 11.5–12.0)
  TTFT P95:                  482 ms      (spread: 461–498 ms)
  TPOT mean:                 17.6 ms     (spread: 17.1–18.1 ms)
  Total latency P95:        4.31 s      (spread: 4.20–4.44 s)
  Accuracy on ref set:       0.874       (floor 0.870, PASS)

Verdict: PASS — sustains 11.8 req/s within SLA, accuracy above floor.
```

A team that produces this report per stack and per candidate model has a decision-quality serving benchmark. A team that produces "tokens per second on Llama-3 was 500" has produced a headline; nobody can act on it.

## Two frequent confusions

### Confusion: benchmark throughput vs. production capacity

A stack that benchmarks at 12 req/s per node is not a stack that will serve 12 req/s per node in production, because production has real cost budgets, real reliability targets (you provision for peak, not for average), real failure and re-try behavior, and real bursty traffic. The benchmark number is the *capacity ceiling* under the specified conditions; the production planning number is that ceiling with capacity-planning overhead (typically 40–70% of ceiling for utilization budgets that respect the P99 latency SLA under bursty arrivals). Chapter 5 of Kohavi et al. is the analog for A/B ramp planning; the standard SRE literature covers the capacity-planning side. The benchmark alone is not the production plan.

### Confusion: MLPerf publications vs. custom benchmarks

MLPerf submissions are audited, peer-reviewed, and follow rigid submission templates; they are useful as reference points for cross-vendor comparability. But they are on *MLPerf's workloads*, not yours. A cross-vendor MLPerf comparison tells you which vendor's H100 stack is faster on the Llama-3.1-405B benchmark workload; it does not tell you which is faster on your production chat workload with your prompt template and your tool-call pattern. The two are related — vendors that do well on MLPerf tend to have well-tuned stacks — but not substitutable. The MLPerf number is a directional signal; the number that drives your infrastructure decision is your own benchmark, structured with MLPerf's methodology.

## Guidance for the eval author

- **Four numbers, always.** TTFT, TPOT, throughput, accuracy against a floor. Any serving-benchmark reporting that omits one of the four is under-specified.
- **The right scenario for the workload.** Offline for batch, Server for interactive chat, Single-Stream for edge / single-user. Do not quote Offline numbers for a Server product.
- **SLA-anchored latency in the Server scenario.** Pre-commit the TTFT / TPOT thresholds; a run that violates them is not a data point, it is a failure.
- **Accuracy floor guards optimization claims.** Every throughput number is quoted alongside the accuracy that the same configuration produced.
- **Reproducibility manifest is not optional.** Hardware, software, weights, workload, warm-up, run count. Without it, the number is not portable.
- **Vendor numbers are directional; run your own on your workload.** MLPerf comparability across submissions is real; comparability to your production workload is not automatic.
- **Wire the benchmark into the release process.** Serving numbers are release artifacts; regression on the benchmark is treated with the same discipline as regression on quality.

## Summary

Serving-time performance is a distinct axis from capability, quality, and safety measurement, and it belongs alongside them in every release decision. The MLPerf Inference methodology — four scenarios (Offline, Server, Single-Stream, Multi-Stream) with four numbers (TTFT, TPOT, throughput, accuracy floor) — is the reference discipline for measuring it in a way that stakeholders and other teams can trust. The right scenario matches the product surface (Server for interactive chat, Offline for batch, Single-Stream for edge); the accuracy floor keeps throughput claims honest by requiring the reported number to come from a configuration whose task accuracy is above a pre-committed threshold; the reproducibility manifest (hardware, software, weights, workload, warm-up, run count) is what makes the number portable across teams and time. Vendor headline numbers are directional signals, not decision inputs; internal benchmarks structured with MLPerf-style discipline give infrastructure decisions the same defensibility that the offline regression suite gives quality decisions. This closes the module: the four altitudes of production eval — offline regression, shadow, A/B, continuous monitoring — plus the two supporting systems — observability and serving benchmarks — compose into the release scorecard that mod-112 assembles into an end-to-end eval system.
