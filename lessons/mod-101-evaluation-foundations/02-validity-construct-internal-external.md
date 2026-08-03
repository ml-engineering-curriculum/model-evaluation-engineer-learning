# Validity: Construct, Internal, and External

Before you ask "how tight is the confidence interval," ask "does this number measure what I claim it measures." That is the validity question. This chapter gives you three frames — construct, internal, and external validity — and shows how to identify the dominant validity threat for a given eval design.

The three-frame vocabulary comes from the experimental-methodology literature (Campbell & Stanley 1963; Cook & Campbell 1979; construct validity in particular from Cronbach & Meehl 1955). ML evals inherit the vocabulary directly, and the failure modes look almost identical to the ones the social-sciences literature has been cataloguing for sixty years.

## Construct validity: does the score measure the construct?

**Definition.** Construct validity asks whether the operationalization (the eval) actually measures the theoretical construct (the thing you say you care about). "Reasoning ability," "helpfulness," "toxicity," "instruction following" are constructs. `GSM8K score`, `Arena win-rate`, `HarmBench refusal rate` are operationalizations.

The construct-validity question is: if the operationalization goes up, does the construct go up?

**Threats.**

- **Construct underrepresentation.** The eval covers only part of the construct. A "math reasoning" eval built entirely from grade-school word problems captures none of the algebra, geometry, or proof-writing that "math reasoning" should include. Small operational lift, no construct lift.
- **Construct-irrelevant variance.** The eval measures the construct *plus* something else, and improvements can come from the something-else. LLM leaderboards where the same model, with a different chat template or a different system prompt, moves several points are picking up prompt-format sensitivity as construct-irrelevant variance. If you swap templates and the ranking flips, the ranking was measuring the prompt, not the model.
- **Contamination.** The eval items appeared in training. The score now measures memorization plus capability, not capability alone. This is the construct-validity failure mode that dominates modern LLM benchmarks; mod-102 covers detection and mitigation in depth.
- **Scorer construct drift.** An LLM judge scoring "helpfulness" that has learned to reward verbose, well-formatted answers is measuring "verbose formatting" plus "helpfulness." mod-105 covers position, length, and self-preference bias in judges.
- **Rubric ambiguity.** A human rubric for "harmful" without decision rules produces inter-annotator agreement in the 0.4–0.6 kappa range; the "harm rate" is then partly a measurement of the individual annotators. mod-106 covers IAA and adjudication.

**How to notice.** Ask three questions of any eval:

1. If a model's score on this eval doubled tomorrow, would we sincerely believe its ability at the construct doubled? If no, the eval is a proxy — say what the proxy is.
2. What could plausibly move this score *without* moving the construct? Chat template? Sampling temperature? Whether the scorer sees the reference before scoring? These are the construct-irrelevant-variance channels.
3. Is any part of the eval set in any plausible pretraining corpus? For public benchmarks with n-gram-searchable items, treat the answer as "yes" unless you have a decontamination story.

## Internal validity: are the comparisons inside the eval sound?

**Definition.** Internal validity asks whether observed differences within the eval are attributable to the manipulation of interest — here, "which system produced the output" — rather than to a confounder.

For an eval, "internal validity" mostly means: when we say model B scored higher than model A, is that because of model B, or because of something else that co-varied with the choice of model?

**Threats.**

- **Test-set leakage across systems.** Model A was tuned against the eval set (or a superset of it); model B was not. B's lower score is partly a measurement of A's leakage. Dev-set-as-test is the classic case.
- **Decoding-parameter drift.** A is run at `temperature=0`, B at `temperature=0.7`. The difference now measures the decoding regime as much as the model. Publish and freeze decoding settings per eval.
- **Prompt drift.** A gets the "official" chat template; B gets a hand-tuned prompt because the team wanted B to look good. Even small template changes move scores multiple points. Freeze the prompt with the eval version (mod-102).
- **Scorer changes mid-run.** The judge model was updated between running A and running B. The comparison now measures a mixture of model change and judge change. Pin judge model + judge prompt + judge decoding.
- **Non-independence within the eval.** Multi-turn dialogues where each turn is scored separately are not independent samples of the model's ability; a bad turn 2 is often caused by a bad turn 1. Treating turns as independent inflates the effective N and shrinks the CI. Bootstrap should resample dialogues, not turns.
- **Selection into the eval set.** If items are added to the benchmark based on which examples the current model gets wrong, the benchmark is now co-adapted with the model and does not measure a fixed target. Freeze the item set before scoring, or use a held-out canary set.

**How to notice.** For every "A > B" claim in a report, ask what *else* changed between the A run and the B run. If more than the model changed, the difference is a mixture.

## External validity: does the result generalize?

**Definition.** External validity asks whether the finding generalizes beyond the specific eval set, population, and setting used.

An eval is always a sample from something. External validity asks what that something is, and whether the deployment surface is inside it.

**Threats.**

- **Sampling frame ≠ deployment surface.** A code eval built from Python competitive-programming problems does not generalize to enterprise Java refactoring, even if both are "coding." A safety eval built from English adult-user prompts does not generalize to non-English or to children's traffic.
- **Overfitting to the benchmark community.** After a benchmark becomes popular, models are increasingly trained with it in the loop (contamination, targeted fine-tuning, prompt-engineered pipelines). Later scores on the same benchmark measure benchmark-specific optimization more than general capability. See Recht et al. (2019), "Do ImageNet Classifiers Generalize to ImageNet?" for the canonical demonstration in vision — new independently collected test sets showed 11–14 point accuracy drops on state-of-the-art ImageNet models, entirely attributable to distribution shift rather than adaptive overfitting.
- **Timing.** Web-scraped evals from year `Y` do not generalize to prompts about events in year `Y+2`. Any eval whose ground truth is time-varying (news, prices, sports, current events) has a shelf life.
- **Population narrowness.** Human-eval panels of ten English-speaking crowdworkers do not generalize to the global user base. This is a construct-*and*-external issue: the construct becomes "what those ten workers prefer" and does not extend.
- **Distribution mismatch between the eval and production traffic.** Model performance on a benchmark of clean, well-formed questions does not predict performance on the malformed, multi-intent, sometimes hostile queries that hit a real production endpoint.

**How to notice.** For every headline number, write one sentence about the population, the timing, and the setting it was measured in. If that sentence differs from the deployment setting, the external-validity threat is the gap.

## Statistical conclusion validity (a fourth frame, briefly)

The Cook & Campbell framework has a fourth type — statistical conclusion validity — asking whether the statistical inferences drawn from the data are sound: correct test choice, sufficient power, appropriate multiple-comparison correction, honest reporting of uncertainty. Chapters 3–6 of this module are entirely about statistical conclusion validity. We separate it out because the fix is technical (use the right test) rather than a design choice (change the eval).

## Picking the dominant threat for a given design

Not every eval has all four threats equally. A quick heuristic:

| Eval shape | Dominant validity threat |
|---|---|
| Public LLM benchmark (MMLU, GSM8K, HumanEval) on a frontier model | **Construct validity via contamination.** Assume leakage until proven otherwise. |
| Internal benchmark hand-built by the team that ships the model | **Internal validity via co-adaptation.** The eval was authored knowing what the model does well. |
| LLM-as-judge win-rate against a strong baseline | **Construct validity via judge bias** (position, length, self-preference) *plus* internal validity if the judge and challenger share pretraining. |
| Human-eval Likert study, n=50, one panel | **External validity via population narrowness** *plus* statistical conclusion validity (CI is wider than the reported difference). |
| Safety refusal-rate on a fixed HarmBench version | **External validity via distribution mismatch** — real jailbreaks evolve; benchmark items age fast. |
| Production A/B on live traffic | Usually strong on external validity; watch **statistical conclusion validity** (were the CIs computed for the actual comparison, were slice results corrected for multiplicity). |

The heuristic is not a diagnosis; it is a starting hypothesis. Chapter 7 shows how to interrogate a specific report and land on the specific dominant threat.

## A worked example

*"On our internal 'helpfulness' benchmark, Model B beats Model A 62% of the time (n=200). We recommend shipping Model B."*

Threats to check, in decreasing order of typical severity for this shape of eval:

1. **Construct validity.** Is "helpfulness" the LLM-judge's operationalization, or is there a human anchor? If judge-only, position bias and length bias are likely construct-irrelevant variance channels — verify the judge was run with swap-and-average, and check whether B's outputs are systematically longer than A's.
2. **Internal validity.** Was the same set of 200 prompts used for both models, or were they resampled? Were they run at the same decoding settings, through the same prompt template? Was the judge the same across runs?
3. **Statistical conclusion validity.** 62% of 200 is 124 wins, 76 losses. The Wilson interval on 124/200 is roughly `[0.55, 0.69]`. If the null of interest is "no preference" (50/50), that is significant. If the null of interest is "the current champion wins 60% of the time and we need clearly better," a 62% point estimate is nowhere near clear. State the null.
4. **External validity.** Where do those 200 prompts come from? User traffic sampled last quarter? A hand-picked "helpfulness suite"? If the deployment surface has drifted (new features, new user segments), the 62% may not survive contact with live traffic.

None of these questions require running any new numbers. They are things you ask before you look at the number. That is the point of the validity frame — it is a checklist you apply to the *design* of the eval, upstream of any statistical machinery.

## Summary

Validity is the question of whether the number means what you say it means, and it splits into three frames: construct (does the eval measure the construct at all), internal (are within-eval comparisons attributable to the system), external (does the result generalize to the deployment). Statistical conclusion validity is a fourth frame that the rest of this module addresses directly. For a given eval design, most of the validity risk concentrates in one frame; the heuristic table above is a starting point, and Chapter 7 shows how to land on the specific threat for a specific report.
