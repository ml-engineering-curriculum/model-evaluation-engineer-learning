# Trajectory Scoring and Budget Tracking

An agent trajectory is an ordered log. Each row is either the assistant's next message (with an optional tool call), a tool result, or the terminal answer. A trajectory scorer's job is to consume that log and emit a structured verdict — not one number, but a small vector of numbers that together describe both *how well the agent solved the task* and *what it cost*. This chapter is about the shape of that vector, the four kinds of signal that go into it, and the reporting discipline that keeps a trajectory-level number from silently degrading into "did the agent's last message match the target."

The chapter is deliberately implementation-heavy. Trajectory scoring is not a metric formula; it is a small pipeline (parse the log, apply per-step checks, apply final-answer check, compute budget rollups, aggregate). Every design choice inside that pipeline has a matching failure mode in the wild, and the exercise-01 rubric will test them one by one.

## Anatomy of a trajectory

A trajectory has, at minimum:

- A **task descriptor**: the prompt, the tool schemas the agent was given, the start state, the success criterion, and the budget caps (max steps, max tokens, max wall-clock, max dollars).
- A **message list** in temporal order: alternating `assistant` and `tool` roles, where each assistant message may carry zero or more tool calls, and each tool call has a matching tool-result message immediately after.
- A **terminal event**: either the assistant produced a final answer (declared, via a terminating tool like `submit(...)` or by matching a stop condition), or the agent hit a budget cap, or the harness aborted (crash, provider error, sandbox failure).
- **Per-step metadata**: for each assistant turn, at minimum the number of prompt tokens and completion tokens; ideally also the wall-clock latency, the estimated dollar cost, and the model / temperature / seed used.

A concrete example — a single trajectory from a made-up "look up the current weather and format it as JSON" task, rendered as JSON Lines so the shape is legible:

```jsonl
{"role": "system", "content": "You have tools: web_search, http_get, submit."}
{"role": "user", "content": "Return the current temperature at 40.7,-74.0 as {\"temp_c\": <number>}."}
{"role": "assistant", "content": "I'll look up the weather.", "tool_calls": [{"name": "web_search", "arguments": {"q": "current temperature 40.7,-74.0"}}], "usage": {"prompt": 220, "completion": 25}, "latency_s": 0.9}
{"role": "tool", "name": "web_search", "content": "[{\"title\": \"api.weather.gov/points/40.7,-74.0\", ...}]"}
{"role": "assistant", "content": "I'll query the NWS API.", "tool_calls": [{"name": "http_get", "arguments": {"url": "https://api.weather.gov/points/40.7,-74.0/forecast"}}], "usage": {"prompt": 480, "completion": 30}, "latency_s": 1.1}
{"role": "tool", "name": "http_get", "content": "{\"properties\": {\"periods\": [{\"temperature\": 61, \"temperatureUnit\": \"F\"}]}}"}
{"role": "assistant", "content": "61 F is 16.1 C.", "tool_calls": [{"name": "submit", "arguments": {"answer": "{\"temp_c\": 16.1}"}}], "usage": {"prompt": 720, "completion": 20}, "latency_s": 0.8}
{"role": "terminal", "final_answer": "{\"temp_c\": 16.1}", "reason": "submit"}
```

Every scorer input in this module has, morally, that shape. Real formats (Inspect's `EvalSample`, SWE-bench's prediction JSON, OpenAI's Responses object, WebArena's action trace) differ in names and nesting but not in content. Chapter 5 shows how Inspect materializes it.

## The four signals a trajectory scorer must emit

A defensible trajectory scorer emits a vector, not a scalar. The four coordinates:

### 1. Final-answer correctness

The mod-107 signal, unchanged. The trajectory reached a terminal state with a final answer; the scorer decides whether that answer is correct against the task's success criterion. The success criterion can be:

- **Exact-match / regex-extract** against a target string. Works for tasks with a canonical answer (GAIA's "return the answer in `<value>` tags"; SWE-bench's "produce a patch that makes the hidden tests pass" once the patch is applied and tests run).
- **Executable check.** A predicate function receives the final state and returns a bool. WebArena uses this — the success criterion for "buy the cheapest red shirt" is a function that inspects the resulting cart, not a string match on the assistant's message.
- **Model-graded / rubric-based.** A judge (mod-105) reads the final answer and grades against a rubric. Used when the task's success criterion is genuinely subjective (open-ended writing, explanatory answers).

The verdict is binary, categorical (`pass` / `fail` / `partial` / `no_answer`), or a rubric score. All three shapes are legitimate; the scorer must declare which.

### 2. Per-step tool-call correctness

For every assistant turn that called a tool, the scorer can ask:

- **Was the tool choice appropriate?** Given the state at that step, was the called tool the right one, or would a different tool have been better? This can be judged automatically only when there is a reference solution or an obvious per-step correct action — most WebArena tasks and most SWE-bench issues do not have a single canonical action sequence, so an automatic per-step judgement will over-penalize valid alternate paths. This is where the mod-105 judge (Chapter 4 of this module) or a human rubric is the correct instrument, not a string comparison.
- **Were the arguments well-formed?** Tool-call schemas are typed; the arguments should validate against the schema. A malformed argument (`{"url": null}`, or a JSON parse failure on the function-call args) is scorable *deterministically*. Call this **tool-call validity rate**: fraction of tool calls whose arguments pass JSON-schema validation.
- **Did the tool actually execute successfully?** A tool result carrying a non-error payload is a different signal from an error. Call this **tool-execution success rate**: fraction of tool calls that returned without an error status. High validity + low execution success is a specific diagnostic: the model formats calls correctly but keeps invoking them on inputs that the tool rejects.
- **Was the sequence non-redundant?** A trajectory that calls the same read-only tool with the same arguments 5 times in a row is doing work that has no signal value; the model looped. Call this **redundant-call rate**: fraction of tool calls that repeat a prior call's `(name, canonical(arguments))` within the same trajectory.

Report all four as separate numbers, not one aggregate. Each is diagnostic of a different failure mode and shipping them as a single "trajectory quality" score erases the signal.

### 3. Budget consumption

Every trajectory has resource costs. The scorer records:

- **Step count.** Number of assistant turns until termination. Report both the *distribution* across the eval and the fraction of tasks that hit the step cap without terminating.
- **Token usage.** Prompt tokens, completion tokens, and total, per trajectory. For providers that charge different prices, report cost in dollars alongside — cost is the artefact the model author actually optimizes against.
- **Wall-clock latency.** End-to-end trajectory time, including tool execution. Latency is *user-experienced* cost; even a cheap trajectory that takes 60 seconds is a bad trajectory for many product surfaces.
- **Tool-call count.** Distinct from step count when a single assistant turn calls multiple tools.

The reporting rule: **never report accuracy without a budget.** A model that scores 55% on WebArena at an average of 8 steps per task is a different model from one that scores 55% at 40 steps per task. If you cite one number without the other, you have given the reader an incomplete comparison.

### 4. Termination reason

The trajectory ended because:

- `submit` (or the model's terminal-tool equivalent): the agent decided it was done.
- `budget_exhausted`: hit the step / token / time cap. The scorer treats this as `no_answer` (the model did not produce an answer) or as `fail_by_timeout` (a distinct bucket you should report separately from wrong-answer).
- `harness_error`: sandbox crashed, tool infrastructure failed, provider errored. This is a *harness* failure, not a model failure, and should be excluded from accuracy numerators when reporting or explicitly categorized as "unknown."
- `abort_by_scorer`: some scorers abort a trajectory early if the agent takes a destructive action (deleted the wrong file, sent an out-of-scope email); usually only appears in agent safety evals (mod-109), but the plumbing lives here.

A scorer that lumps `budget_exhausted` and `wrong_answer` into a single "failure" bucket loses the ability to distinguish "the model would have solved it with more budget" from "the model was going to fail regardless." Report the termination-reason distribution alongside accuracy.

## The scorer as a function

Written as a Python signature, a trajectory scorer looks approximately like:

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable, Sequence

class Termination(str, Enum):
    submitted = "submitted"
    budget_exhausted = "budget_exhausted"
    harness_error = "harness_error"
    abort_by_scorer = "abort_by_scorer"

class Verdict(str, Enum):
    pass_ = "pass"
    fail = "fail"
    partial = "partial"
    no_answer = "no_answer"

@dataclass
class TrajectoryScore:
    verdict: Verdict
    final_answer_score: float           # 0.0 - 1.0
    tool_call_validity_rate: float      # 0.0 - 1.0
    tool_execution_success_rate: float  # 0.0 - 1.0
    redundant_call_rate: float          # 0.0 - 1.0
    steps: int
    prompt_tokens: int
    completion_tokens: int
    wall_clock_s: float
    dollars: float
    termination: Termination
    notes: list[str] = field(default_factory=list)

def score_trajectory(
    trajectory: Trajectory,
    task: TaskSpec,
    correctness_check: Callable[[Trajectory, TaskSpec], tuple[Verdict, float]],
) -> TrajectoryScore:
    ...
```

Two structural rules that keep this honest:

- **The final-answer check is injected.** Do not hard-code exact-match. `correctness_check` is a callable — for one task family it is `re.search(target, final_answer)`; for another it is `run_hidden_tests(patch)`; for a third it is `judge.grade(final_answer, rubric)`. Chapter 4 formalizes the rubric-based version.
- **Per-step rates are computed from the trajectory alone.** Tool-call validity, execution success, and redundancy do not need the target. They are pure trajectory statistics and can be computed on any trajectory including failures.

## Aggregation across the eval

A single-trajectory score is one row. The eval-level report is the aggregate over rows. Three aggregation questions have non-obvious answers.

### How do you aggregate accuracy?

Report the fraction of trajectories with `verdict == pass`, and always report *alongside* the fraction with `verdict == no_answer` (typically = budget-exhausted trajectories) and `verdict == fail` (returned an answer, but wrong). Do *not* fold `no_answer` into `fail` by default — a model that fails-by-timeout on 30% of the eval has a different capability profile from one that fails-by-wrong-answer on 30%, and a step-budget increase can move the first metric but not the second.

Confidence intervals: bootstrap over *trajectories*, not over *steps*. A common bug in agent eval reporting is treating each per-step call as an independent trial and computing a per-step confidence interval — this understates variance because per-step outcomes within a trajectory are highly correlated. The unit of independence is the trajectory (see mod-101 Chapter 2).

### How do you aggregate cost?

Report medians and 95th percentiles alongside the mean. Agent-eval cost distributions are heavy-tailed: a single stuck trajectory with 100 tool calls dominates the mean. A model whose *median* trajectory is cheap and whose *p95* trajectory is 20× the median has a specific failure profile — it usually solves fast, but when it fails, it fails expensively. That profile is invisible if you only report the mean.

The `cost-vs-accuracy Pareto frontier` is the standard visualization: X-axis mean-cost-per-task, Y-axis accuracy, one point per (model, config). A model dominates another on the frontier if it is at least as cheap and at least as accurate. Model comparisons over agent evals should show the Pareto plot; a single scalar "accuracy" without the cost axis is not enough to make a decision.

### How do you aggregate the per-step signals?

The four per-step rates (validity, execution success, redundancy, and the tool-choice rate if you have one) are per-trajectory means already. The eval-level report is the mean across trajectories, plus a per-slice breakdown when the eval has categories (SWE-bench by repository, WebArena by task category). Slice reporting is where per-step signals earn their keep — an eval with 55% accuracy overall might have 92% tool-call validity on `django/django` and 41% validity on `sympy/sympy`, and the diagnostic is "the model does not know the sympy tool surface" rather than "the model is worse at math."

## The budget policy is part of the task definition

A subtle point that catches every first-time agent-eval implementer: **the step / token / time budget is not a scorer-side knob, it is part of the task**. Two evaluations of the same benchmark at different budgets are not comparable. If you run SWE-bench at a 40-step cap and a peer runs it at a 100-step cap, your numbers are on a different task even though the benchmark name is the same. Every published SWE-bench number should carry the budget alongside; every WebArena number should carry the `max_actions` cap alongside.

The rules of thumb:

- **Set the budget from the reference solution, not from cost.** If a well-instrumented reference solver solves the task in 15 steps on average, a 30-step cap is defensible; a 5-step cap is too tight and a 200-step cap is essentially unbounded. Setting the budget from your inference bill is expedient but not defensible — you have coupled the task definition to your funding.
- **Log the fraction of trajectories that hit the cap.** If more than a small fraction of trajectories hit the step cap, the budget is doing scoring work (converting "would have solved eventually" into "failed") that you should surface explicitly.
- **Publish the budget in the reproducibility manifest.** Alongside model version, dataset hash, and harness version, the budget cap belongs in the pin.

## Where the four signals catch bugs the final answer misses

Three concrete diagnostic scenarios where per-step signals surface bugs that a final-answer-only scorer would miss:

- **The model is right for the wrong reason.** Final-answer accuracy is 60%. Tool-call validity is 45%. What happened: the model produces final answers that pattern-match the training-distribution answer to superficially-similar questions and only uses tools when it doesn't already "know" — so it looks 60% accurate but only 45% of its tool calls even parse. On out-of-distribution items the accuracy will collapse. A final-answer-only scorer never flags this.
- **The model brute-forces.** Final-answer accuracy is 70%. Median step count is 6. p95 step count is 45. Redundant-call rate is 0.30. What happened: on 70% of tasks the model solves cleanly; on the remaining 30% it thrashes — calls the same tool repeatedly with slight argument variations until it either stumbles into the right answer or hits the budget. The Pareto plot puts this model far to the right (high cost). Solving that 30% with fewer steps is where the product gain lives.
- **The sandbox is silently broken.** Final-answer accuracy is 35%. Tool-execution success rate is 60%. Termination-reason `harness_error` is 15%. What happened: 15% of trajectories failed because the sandbox crashed. Another 25% consumed tool calls that returned errors because a tool implementation is buggy. The model's true capability is not 35%; the eval instrument is broken. A final-answer scorer reports 35% and buries the sandbox bug in the noise.

Each of these is a real class of finding that shipped-in-the-wild agent-eval reports have made. The pattern is the same: the per-step signals are the diagnostic, and shipping them next to accuracy is nearly free once the scorer is emitting the vector.

## Cost, correctness, and the pass^k idea

SWE-bench Verified and several recent agent-eval reports popularized a variant: `pass^k` (Chen et al. 2021 gave `pass@k` for code eval; the agent-eval variant is sometimes written `pass^k` and pronounced "pass power k"). Run the same task `k` times independently, report the fraction of tasks where *all `k`* runs pass. This is a *consistency* metric, not a capability metric: it punishes non-determinism and lucky one-off successes. A model that scores `pass@1 = 0.60` and `pass^5 = 0.30` is one where roughly half of the "solved" tasks are stochastic near-misses. The reporting practice — report both — comes from the same lineage as the mod-107 pass@k discipline and is cheap to add once you already have multiple epochs (Inspect exposes it directly via `--epochs`).

The dual metric `pass^k` is not a replacement for `pass@k`; a report should carry both plus the cost axis. Under-reporting one hides a lot.

## Summary

A trajectory scorer emits a small vector — final-answer correctness, per-step tool-call validity / execution success / redundancy, and a budget rollup (steps, tokens, dollars, wall-clock) with an explicit termination reason. The budget is part of the task, not the scorer; two runs at different budgets are not comparable. Aggregation is over trajectories with bootstrap CIs and Pareto-frontier plots against cost; heavy-tailed cost distributions require reporting medians and p95s, not just means. Per-step signals are what let a report distinguish "the model got the right answer for the wrong reason," "the model brute-forced" and "the sandbox is broken" from a single final-answer number. The next chapter takes this scorer to SWE-bench — the largest and best-documented agent benchmark — and walks the reproduction pipeline for the published number.
