# Why Agent and Tool-Use Evaluation Breaks Single-Turn Scoring

Every module up to this point has evaluated a single request-response pair. The model produced a string; a scorer graded the string against a target, a rubric, a test suite, or a judge. Even in mod-107's code chapter, where the sandbox runs *generated code* rather than the model itself, the eval is still one turn wide: prompt in, program out, tests grade the program. That shape stops fitting the moment the model is allowed to loop — to inspect its own output, call a tool, observe a result, and decide what to do next.

An agent eval measures a *trajectory*: an ordered sequence of thoughts, tool calls, tool responses, and eventually a final answer (or a decision to give up). The final answer is still gradeable by the tools you already have. But the trajectory is where most agent bugs and most agent capability lives, and if the scorer only ever looks at the last message, it silently agrees to score two very different processes as the same outcome — including scoring "the agent solved this in one shot" and "the agent burned 40 turns, called the wrong tool six times, hallucinated a filename, and stumbled into the right answer" as identical successes. This module is about making that distinction load-bearing.

## What changes when an agent is in the loop

Four things change simultaneously the moment the model can call tools and observe results before answering:

- **The output object grows.** It is no longer a string but a *trajectory* — a list of `(assistant_message, tool_call?, tool_result?)` triples ending with a terminal message. Some frameworks call this a message list; others a "run"; SWE-bench calls it a "prediction"; the OpenAI Responses API calls it a "response with items." The shape is the same: an ordered log of who said what and what they observed.
- **Correctness is multi-axis.** The final answer can be right or wrong (the mod-107 shape). Independently, each intermediate tool call can be correct-choice, correct-arguments, both, or neither. Independently again, the trajectory has a *cost profile* — token count, wall-clock, tool invocations, dollar cost — and a task with an unbounded budget is not the same task as one with a 20-step cap. A trajectory-level scorer has to report each axis without collapsing them into a single scalar.
- **The environment is stateful.** The model's tool calls change the world (write files, click buttons, submit forms, open shell processes). Two runs on the same task start in different states unless the environment is reset. A repeatable eval needs *sandbox isolation*: a fresh, deterministic starting state per sample, so nothing in run *i+1* is contaminated by run *i*.
- **Determinism becomes a design goal, not a default.** The web page you scraped in run 1 changed by run 2. The GitHub issue you opened persists. The subprocess you spawned may still be alive. Without a replay strategy — recorded HTTP traffic, snapshotted filesystems, containerized worlds, seed-controlled tool implementations — you cannot re-run the eval and get the same numbers, which means you cannot A/B two models against each other on the same task.

Every one of these is an implementation detail that the mod-107 code-eval sandbox does not need to worry about (execution is a single subprocess call, the sandbox is torn down after, the model's only "trajectory" is the tokens of the completion). Agent eval inherits everything from the mod-107 sandbox and then adds statefulness, a message loop, and a per-step measurement surface on top.

## The four objects, reprised for agent evals

The four-objects framing from mod-104 (model adapter, task definition, request type, scorer) still applies, but each object gets richer:

- **Model adapter.** Not "generate a completion" but "call the model, parse tool calls, dispatch them, feed results back, loop until terminal message or budget exhaustion." This is the *agent harness* — `basic_agent` in Inspect, the ReAct loop from Yao et al. 2023 (`ReAct: Synergizing Reasoning and Acting in Language Models`), or the vendor-supplied loops (OpenAI Assistants / Responses, Anthropic tool-use, Google GenAI function calling). Chapter 5 walks Inspect's version end-to-end because it is the most transparent about what the loop does; the others are conceptually the same.
- **Task definition.** A start state (repository at a specific commit, browser at a specific URL, filesystem seeded with specific files), a set of available tools (with their schemas and side effects), a task description (natural-language or structured), and — crucially — a *success criterion*. The success criterion is where the interesting design choices live: a hidden test suite (SWE-bench), a browser-state predicate (WebArena), a final-answer match with an executable grader (GAIA), or a partial-credit rubric (Chapter 4).
- **Request type.** Tool-using multi-turn generation. The request type is no longer "loglikelihood" or "generate_until"; it is "run this agent to completion or budget exhaustion and return the trajectory."
- **Scorer.** The scorer now looks at three things and must decide how to aggregate them: the final state / final answer (traditional correctness), the intermediate trajectory (per-step tool-call correctness, order violations, redundant work), and the resource consumption (steps used, tokens spent, wall-clock, dollars). The interesting scorers report all three axes and let the report author combine them with product-appropriate weights; the least-useful scorers report only final-answer accuracy and let the trajectory information rot.

Chapter 2 builds the trajectory scorer end-to-end. Chapters 3 and 4 walk the two canonical benchmarks (SWE-bench, WebArena / GAIA) and the sandbox and partial-credit machinery they require. Chapter 5 shows the Inspect harness pattern that ties the module together and gives you a durable template for internal agent evals.

## Why this module exists as its own thing

The obvious question: why not fold agent eval into the harnesses module (mod-104) or the code-generation chapter (mod-107)? Two reasons.

First, the *unit of measurement* is different. mod-104's harnesses batch one-shot requests over a dataset. mod-107's code chapter runs each sample through a sandbox once. Agent evals run each sample through an interactive loop that may take minutes and may fail in the middle for reasons unrelated to model capability (browser flaked, container OOM'd, tool schema changed). Batch throughput, retry semantics, and partial-run recovery all look different, and the reporting practice differs — you cannot bootstrap over "trials" the way you do over items when a single trial can consume dozens of dollars of inference.

Second, the *failure catalogue* is different. Nearly all mod-107 failure modes are scorer bugs (timeout too tight, sandbox mis-configured, test suite over-tests). Agent-eval failure modes include most of those *plus*: trajectory-shaping (the scorer only looks at the last message and misses that the agent brute-forced), sandbox contamination (state leaks between samples), non-deterministic environments (a WebArena page changed under you), tool-schema drift (the tool the model was trained on has a different signature now), and cost-normalization mistakes (comparing an agent that used 500 tokens per turn against one that used 5,000, on "same task solved" as if they were interchangeable). Every one of those needs its own chapter-worth of discipline.

## Where this module sits in the track

The prerequisites you should have solidified before starting:

- **mod-104 (harnesses).** You know how to register a task, how a solver / scorer factor apart, and how Inspect in particular treats an eval as a program on state. Chapter 5 assumes you can read Inspect code without a refresher.
- **mod-105 (LLM as judge).** The partial-credit rubric in Chapter 4 is an LLM-as-judge applied to trajectories rather than single outputs. Bias controls (position, length, self-preference) still apply; anchor-based rubrics still apply; calibration against humans still applies.
- **mod-107 (generation and sandbox).** The mod-107 sandbox (process, filesystem, network, memory isolation) is a subset of what agent evals need. SWE-bench's per-repository containers are a superset. Chapter 3 assumes the mod-107 sandbox discussion is fresh.

The things this module explicitly does *not* cover:

- **Safety evals for agent capabilities** (autonomy, self-exfiltration, unauthorized action). Those live in mod-109; agent evals under an unrestricted sandbox with real credentials are a different discipline, and the mod-109 chapters cover the risk-framework mapping.
- **Production regression / A/B testing of deployed agents.** mod-110 covers online evaluation for agents that ship to users; this module covers *offline* benchmarks.
- **Full-blown eval-platform engineering** (queueing, cost accounting, artifact storage, multi-tenant sandbox pools). mod-111 covers the platform layer; this module assumes a single-node or small-cluster local run.

## A note on the current landscape

Agent evaluation is the youngest sub-field in this track. The reference benchmarks that anchor Chapters 3 and 4 — SWE-bench (Jimenez et al. 2024, ICLR), SWE-bench Verified (OpenAI / Princeton 2024), WebArena (Zhou et al. 2024, ICLR), GAIA (Mialon et al. 2024, ICLR), AgentBench (Liu et al. 2024, ICLR) — all published in 2023–2024, and the community's implementation practices around them are still moving. Public leaderboards on SWE-bench Verified change methodology from month to month; WebArena's canonical Docker image has been re-released; GAIA's evaluation server has changed URL twice at time of writing. Treat every number in this module as version-stamped: cite the benchmark version, the harness version, the container digest, and the model release, or your reproducibility claim is not a claim.

## Summary

Agent and tool-use evaluation extends the single-turn eval shape to trajectories: the output is now an ordered log of assistant messages, tool calls, and tool results, and correctness is a multi-axis property (final answer, per-step tool-call correctness, cost / latency / step budget). The environment is stateful and non-deterministic by default, so sandbox isolation and deterministic replay become first-class engineering concerns. The four-objects framing survives: the model adapter becomes an agent harness, the task definition includes a start state and success criterion, the request type is a bounded tool-using loop, and the scorer aggregates trajectory-level signals rather than reading only the terminal message. The remaining chapters build each of those pieces: Chapter 2 the trajectory scorer, Chapter 3 the SWE-bench pipeline, Chapter 4 the interactive-sandbox benchmarks (WebArena / GAIA / AgentBench) and partial-credit rubrics, Chapter 5 Inspect's agent harness end-to-end.
