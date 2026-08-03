# Partial-Credit Rubrics for Trajectories

Chapter 4 ended on a coarse-vs-diagnostic split: interactive-sandbox benchmarks report a binary per-task verdict for the leaderboard, but the underlying trajectories carry enough information to say *how close* the agent came, *which subgoal it reached*, and *what kind* of mistake it made. Turning that information into a defensible score is a *partial-credit rubric* — a structured judgement about a trajectory's quality that is finer-grained than pass / fail. This chapter is about designing such a rubric, running it (usually as an LLM-as-judge, sometimes as a human review), and auditing it against a gold-standard set so the rubric's numbers aren't just a mid-level model's tone-matching.

The chapter borrows heavily from mod-105 (LLM-as-judge platforms) — the bias controls, the anchor design, the human-calibration protocol are all the same techniques. The difference is that the *input* to the judge is a trajectory (a message list with tool calls), not a single output string, and the *rubric* has to accommodate the trajectory structure. Anchors, position and length bias, and rater-drift concerns still apply.

## Why a rubric on trajectories at all

Three product-relevant questions that a binary "did the agent succeed?" cannot answer:

- **Which subgoals did the agent reach before failing?** A trajectory that failed at the checkout step is closer to solving a shopping task than one that never logged in. When you're comparing two models, the model whose failures cluster near-success is often the one to invest in.
- **What kind of tool-use mistakes did the agent make?** A trajectory can fail because the agent picked the wrong tool, used the right tool with wrong arguments, called the right tool correctly but didn't parse the result, or produced a plausible-looking final answer that was actually wrong. Each is a different capability gap.
- **How efficient was a successful trajectory?** Two agents can both solve a task, but one takes 4 tool calls and the other takes 24. A rubric that awards partial credit for parsimony captures the efficiency gap that a binary success verdict erases.

Chapter 2 already covered the *automatic* trajectory signals — tool-call validity rate, execution success rate, redundant-call rate, budget consumption. Those are the cheap, always-on numbers. This chapter is about the *rubric-based* signals — the judgements that require reading the trajectory in context and forming an opinion, and therefore require either a human rater or an LLM judge.

## Rubric design: three shapes

Rubrics come in three shapes, in increasing order of design cost and information content.

### Shape 1 — Milestone checklist

Enumerate a small set of ordered milestones the trajectory must reach on the way to success. Each milestone is a boolean predicate on the trajectory state.

```yaml
task: "book a flight from SFO to JFK for tomorrow"
milestones:
  - id: m1_search
    description: "Agent submitted a search with SFO origin, JFK destination, tomorrow's date"
    predicate: "any tool call to search_flights with matching arguments"
  - id: m2_results
    description: "Agent viewed a results list"
    predicate: "tool result from search_flights contained at least one flight"
  - id: m3_selection
    description: "Agent selected a specific flight"
    predicate: "any tool call to select_flight with a flight_id argument"
  - id: m4_passenger
    description: "Agent submitted passenger information"
    predicate: "any tool call to add_passenger with name and DOB"
  - id: m5_confirmation
    description: "Agent completed booking and received a confirmation number"
    predicate: "tool result from confirm_booking contained a confirmation_id"
```

Score: fraction of milestones reached. Cheap to author (5–15 minutes per task template), cheap to grade (predicates are automatic once you write them). This is the right first stop for any partial-credit rubric on structured tasks.

Two failure modes to guard against:

- **Milestones that are too easy.** If every trajectory reaches milestone 1, that milestone is not adding information. Prune milestones that all models reach.
- **Milestones that ordering-cheat.** If a task can be legitimately solved in a different order (search before login vs login before search), an ordered milestone list will unfairly penalize the alternate path. Milestone predicates should be *state-based* ("cart contains item X") not *sequence-based* ("the third tool call was add_to_cart").

### Shape 2 — Multi-dimensional scoring rubric

For open-ended trajectories where milestones don't decompose cleanly, define a small number of dimensions and score each on a bounded scale with anchored labels.

```yaml
dimensions:
  - name: goal_alignment
    description: "How well the agent's actions serve the stated goal"
    scale: 0-4
    anchors:
      0: "Agent misunderstood the goal entirely; actions unrelated"
      1: "Agent partially grasped the goal; some actions on-goal, most drift"
      2: "Agent understood the goal; actions mostly on-goal with some detours"
      3: "Agent understood and pursued the goal; minor inefficiency"
      4: "Agent pursued the goal directly and efficiently"
  - name: tool_use_quality
    description: "Correctness of tool choices and arguments"
    scale: 0-4
    anchors:
      0: "Wrong tools, malformed arguments, ignored errors"
      1: "Frequent tool-use errors that block progress"
      2: "Correct tool choices but noisy arguments or missed error signals"
      3: "Mostly correct tool use with a small number of avoidable retries"
      4: "Concise, correct, well-argued tool use throughout"
  - name: recovery_from_error
    description: "How the agent handled tool errors and unexpected results"
    scale: 0-4
    anchors:
      0: "Ignored errors or looped on the same failing call"
      1: "Recognized errors but chose ineffective retries"
      2: "Recovered from some errors, gave up on others"
      3: "Recovered from most errors with reasonable strategies"
      4: "Handled every error observed cleanly and productively"
```

Score: mean across dimensions, or reported per-dimension without aggregation. This is the mod-105 anchored-rubric pattern applied to trajectories. The anchor discipline is the same — write descriptions that a rater can distinguish, do a pilot round with humans, refine the anchors that produce disagreement.

### Shape 3 — Free-form judge critique

For research-y evaluation where you want a qualitative read of what the agent did, ask a judge model (or human) to produce a short critique per trajectory, then extract structured features from the critique.

```yaml
prompt: |
  Read the following agent trajectory that attempted to solve the task.
  Produce a 3-sentence critique that answers:
  1. Did the agent solve the task? (yes / partial / no)
  2. If not fully solved, what was the primary failure mode?
    Choose from: misunderstood_goal, wrong_tool_choice, wrong_arguments,
    ignored_tool_error, hallucinated_result, ran_out_of_budget, other.
  3. What was the strongest positive of the trajectory, if any?
```

Post-process the critique into a categorical failure-mode label + a boolean solved flag. This shape scales worst (each trajectory costs a judge call and the critique text is expensive to store and audit) but produces the richest error taxonomy — you can slice the eval by failure mode and see "the model fails-by-hallucination on 22% of trajectories and fails-by-budget on 8%," which is diagnostic in a way that a scalar score is not.

The three shapes are complementary; a mature agent-eval report often has all three, computed per trajectory, and rolled up per-task-family.

## The judge, and its bias controls

Almost every partial-credit rubric on trajectories is run as an LLM-as-judge, because human labelling at trajectory scale is prohibitive (a trajectory can be 40+ messages; labelling 500 trajectories against a 3-dimension rubric is a full week of a rater's time). Every mod-105 discipline applies:

- **Anchor your scale.** Never ask the judge for "how good was this trajectory" on a 0–10 scale without anchors. The judge will regress to 6-7 and disagreement with humans will be catastrophic. Anchor every scale point with a concrete description; validate that a small human panel agrees on the anchor placement.
- **Position bias.** If your rubric compares two candidate trajectories pairwise (rare for agent evals but occasionally useful for A/B model comparisons), randomize which trajectory is A and which is B. Report position bias as a separate slice.
- **Length bias.** Judges reliably score longer trajectories higher on "thoroughness" dimensions and lower on "efficiency" dimensions than a human would. If the rubric has both, both biases apply. Include an explicit anchor for "concise but complete" so the judge distinguishes efficient from lazy.
- **Self-preference.** A judge that is the same model as the agent will systematically over-rate the agent's trajectories. If you can afford it, use a different judge model from the agent; if you cannot, log the pair explicitly and calibrate for it (see below).
- **Verbosity manipulation.** Agents can pad their trajectories with helpful-sounding narration that a naive judge treats as evidence of care. Include an anchor that flags "excessive narration without corresponding action" as a negative.

## Auditing the rubric against human gold

The rubric is worthless if it disagrees with human judgement. The audit protocol:

1. **Sample.** Pick a *stratified* sample of trajectories — some the automatic scorer marked success, some it marked failure, some at every budget level, some from every task category. 40–100 trajectories is enough for a stable audit at trajectory scale.
2. **Human-label.** Have 2–3 humans independently score every trajectory in the sample against the rubric. Compute inter-annotator agreement (mod-102 Chapter 4): Cohen's κ for pairwise, Fleiss' κ or Krippendorff's α for multi-rater. If human agreement is low (κ < 0.5), the rubric is under-specified — fix the anchors before running the judge.
3. **Judge-label.** Run the LLM judge over the same sample.
4. **Compare.** For each dimension, compute judge-vs-human agreement (κ) and a per-anchor confusion matrix. The killer diagnostic is not the overall κ but the confusion matrix — a judge that systematically confuses anchors 2 and 3 is a judge whose numbers are noisy at the 2–3 boundary and useless if that boundary is where your models cluster.
5. **Threshold.** Decide the minimum judge-vs-human agreement you will accept per dimension. mod-105 gives the numbers: κ > 0.6 for a decision-quality judge; κ > 0.4 for a screening judge that will be re-audited before publication. Below 0.4, the judge is not usable at all for that dimension.

The audit is not a one-time gate. Rubric drift and judge drift both happen: an anchor's meaning creeps as raters see edge cases, a judge model's behavior shifts across releases. Re-audit every quarter, and any time the judge model is upgraded.

## Turning the rubric into a scorer

Once the rubric is designed, audited, and calibrated, wiring it into the Chapter 2 trajectory-scorer is mechanical:

```python
def rubric_scorer(trajectory: Trajectory, task: TaskSpec) -> RubricScore:
    milestone_frac = milestone_check(trajectory, task.milestones)
    judge_response = judge.grade(
        rubric=task.rubric_prompt,
        trajectory=trajectory,
        task_description=task.description,
    )
    return RubricScore(
        milestone_fraction=milestone_frac,
        dimension_scores=judge_response.dimensions,   # {name: 0-4}
        failure_mode=judge_response.failure_mode,     # from critique
        judge_model=judge.model_id,
        judge_prompt_hash=judge.prompt_hash,
    )
```

Two structural rules:

- **Log the judge model and prompt hash.** A rubric score without those is not reproducible. If you change the judge or the prompt, the score changes, and you cannot compare cross-version.
- **Emit multiple dimensions.** Do not average dimensions into a single number and throw the components away. The dimensions are the diagnostic; the average is the summary. Report both.

## Where the rubric approach shines and where it fails

**Shines.** Free-form trajectories with no clean environment-state predicate (open-domain research, multi-file code review, cross-tool orchestration). Diagnosing *how* an agent fails, not just whether it does. Reporting an efficiency dimension alongside a correctness dimension for the same trajectories.

**Fails.** Anywhere a deterministic, automatable success criterion exists — you should always prefer the deterministic scorer for the primary metric and use the rubric for the *secondary* diagnostic dimensions. A rubric-only scorer on SWE-bench would be strictly worse than SWE-bench's test-suite grader: the tests are the ground truth and no LLM judge is going to beat them at correctness attribution. A rubric-only scorer on WebArena would lose the environment-state check that catches "the agent claimed success but the cart is empty" trajectories.

The failure mode of over-using rubrics: it is very easy to build an agent eval where the primary metric is a judge-derived score, the judge silently degrades over releases, and by the time you catch the drift the numbers no longer mean anything. The mod-105 discipline — pin the judge, calibrate against humans, re-audit — is not optional for anything that lives in a leaderboard.

## Summary

A partial-credit rubric turns a trajectory into a small vector of judgements — milestone completion, dimension scores, a critique-derived failure mode — that a binary success verdict cannot express. Three rubric shapes cover the space: milestone checklists for structured tasks, multi-dimensional anchored rubrics for open-ended tasks, and free-form judge critiques for research-y evaluation with structured post-processing. Every rubric runs as an LLM-as-judge inheriting the full mod-105 discipline: anchor the scale, control for position / length / self-preference / verbosity, calibrate against a human-labelled stratified sample, and re-audit on a schedule. Rubric scores complement automatic scorers; they are the primary metric only when no deterministic success criterion is available, and even then they are logged with judge model and prompt hash so a reader can reproduce them. Chapter 6 (final chapter) puts the whole module together by walking Inspect's agent harness — dataset, solver, sandbox, custom trajectory scorer — end-to-end, giving you the template your internal agent evals will look like.
