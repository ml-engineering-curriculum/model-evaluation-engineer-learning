# Fairness Impossibility Results and Tooling

Chapter 5 laid out the standard fairness definitions and how to measure them. This chapter answers the question that comes next in every real audit: *why can't we satisfy all of them?* Two independent 2017 results — Chouldechova, and Kleinberg, Mullainathan, & Raghavan — showed that when the base rates of the outcome differ across groups, the standard fairness definitions are mutually incompatible. The impossibility is not an artifact of one specific method or one specific measurement; it is a theorem about the joint conditional probabilities and a fact about the world you are measuring.

Knowing the theorem changes what a good fairness report looks like. Instead of "we satisfy fairness definition X," a defensible report names *which* trade-off was taken, *why*, and *what evidence* justifies choosing that trade-off over the alternatives. This chapter frames the impossibility result and then walks through how to compute the standard fairness numbers with Fairlearn and Aequitas — the two most common Python libraries — end-to-end.

## The impossibility theorem, informally

Chouldechova (2017) proved: if the base rate `P(Y = 1 | A = a)` differs across groups `a`, then a classifier cannot simultaneously satisfy **predictive parity** (equal PPV across groups), **equal FPR**, and **equal FNR** — except in the trivial cases of a perfect classifier or when the base rates happen to be identical.

Kleinberg, Mullainathan, and Raghavan (2017) independently proved a closely-related result: for any two groups with different base rates, a classifier that satisfies **calibration within groups** cannot simultaneously satisfy **balance for the positive class** (equal average score for true positives across groups) and **balance for the negative class** (equal average score for true negatives across groups).

Both papers frame the same underlying fact: once base rates diverge, the definitions in Chapter 5 pull against each other. Fixing the model to satisfy one either mechanically violates another or requires the base rates to be equal (which they are not, or you would not be here).

**Why this is not a modelling problem.** The impossibility is arithmetic. It applies to any classifier — logistic regression, gradient-boosted trees, deep nets, human decision-makers. Improving the model reduces the *magnitude* of the disparities but does not remove the trade-off; only a perfect classifier or equal base rates does that.

## The COMPAS worked example

The impossibility result is easiest to see in the COMPAS controversy, since it was the concrete case that motivated both papers.

- ProPublica (Angwin et al. 2016) showed that COMPAS scores had a higher false-positive rate for Black defendants than for white defendants. Under the equalized-odds framing (Chapter 5), COMPAS was unfair.
- Northpointe (Dieterich et al. 2016) showed that COMPAS satisfied predictive parity — the same score meant approximately the same recidivism rate regardless of race.
- Both were arithmetically correct on the same data. The base recidivism rate differed by race in the underlying data.
- Chouldechova's impossibility theorem said: given the differing base rates, you *cannot* have both predictive parity and equal FPR. Northpointe chose the former; ProPublica argued for the latter. There is no combination of hyperparameters that gets you out of the trade-off; the choice is a policy question, not a technical one.

A defensible report acknowledges the choice explicitly. "We chose predictive parity because [reason]; we measured the resulting FPR disparity and it was [X], which we accept because [reason]." The alternative — pretending there was no choice — is what allowed both sides in the COMPAS debate to talk past each other for years.

## Practical consequences for the eval engineer

You will be asked to satisfy multiple fairness definitions simultaneously. When base rates differ, the answer is that you cannot, and the request needs to be reframed.

Three concrete responses:

1. **Pick the primary metric and justify it in writing.** Which definition matches the harm you are trying to prevent? A resource-allocation setting where missing a positive is the harm (school admission, loan approval) favors equal opportunity or demographic parity; a screening setting where a false positive is the harm (fraud investigation, content moderation) favors equalized FPR; a decision-support setting where the score is consumed as a probability by a downstream tool favors calibration within groups. The choice is the report's central design decision.
2. **Report the other definitions honestly.** Even after choosing a primary metric, publish the numbers on the others so a reader can see the trade-off you made. A report that satisfies predictive parity while hiding the FPR disparity is what the COMPAS episode was arguing about.
3. **Consider whether the base rates are legitimately different or the result of historical inequity.** If a base rate difference is itself the artifact of prior discrimination (e.g. differential enforcement leading to differential arrest rates), then measuring against the observed base rate hard-codes the inequity into the model. This is where the fairness question crosses over into policy; the eval engineer surfaces the choice and the accountable owner makes it.

## Mitigation strategies and their costs

Fairness *mitigation* — actively reducing disparities in the classifier — sits downstream of measurement. The measurement discipline in Chapters 5–6 is the entry gate. This section is a quick catalog of what mitigation looks like and its downsides; the eval engineer's job is often to *measure the effect of a mitigation* rather than to run it.

- **Pre-processing.** Reweighting or resampling training data so the training distribution has less disparity. Kamiran & Calders (2012) and others. Simple, cheap, hard to prove effective in isolation.
- **In-processing.** Adding fairness constraints to the training loss. Fairlearn's `ExponentiatedGradient` and `GridSearch` reductions (Agarwal et al. 2018) are the canonical implementations. Requires access to training. Cost: some accuracy typically.
- **Post-processing.** Adjusting the decision threshold per group. Hardt et al. (2016) proposed group-specific thresholds achieving equalized odds by construction. Cheap to implement. Cost: per-group thresholds are often a legal / policy problem, and in some jurisdictions they may be legally impermissible (US law generally disallows using protected attributes in the decision function, though this depends on domain and enforcement).
- **Rejection.** For low-confidence predictions in the "gray zone," defer to a human. Reduces disparities by removing the low-confidence decisions that most disparately impact one group. Cost: reviewer capacity.

**Mitigation caveats.**

- Every mitigation trades accuracy for parity. Publish the trade curve, not just the mitigated model.
- Mitigation on the training set can shift disparities to other axes (e.g. equalizing race disparities but worsening age disparities). Re-run the full measurement after mitigation.
- Group-specific thresholds create their own fairness concerns and can be legally sensitive; consult counsel before shipping them.

## Fairlearn: what it computes and how

[Fairlearn](https://fairlearn.org/) is the Microsoft-maintained Python library that is the closest thing to a standard for fairness measurement and mitigation in scikit-learn-shaped pipelines.

**Measurement primitive: `MetricFrame`.**

```python
from fairlearn.metrics import (
    MetricFrame,
    selection_rate, true_positive_rate, false_positive_rate,
    demographic_parity_difference, demographic_parity_ratio,
    equalized_odds_difference, equalized_odds_ratio,
)
from sklearn.metrics import precision_score, recall_score

metrics = {
    "selection_rate": selection_rate,
    "recall": recall_score,
    "precision": precision_score,
    "tpr": true_positive_rate,
    "fpr": false_positive_rate,
}

frame = MetricFrame(
    metrics=metrics,
    y_true=y_test,
    y_pred=y_pred,
    sensitive_features=A_test,
)

frame.by_group      # metric per group; pandas.DataFrame
frame.overall       # overall metric
frame.difference()  # max group difference for each metric
frame.ratio()       # min/max ratio for each metric

dp_diff = demographic_parity_difference(
    y_true=y_test, y_pred=y_pred, sensitive_features=A_test,
)
eo_diff = equalized_odds_difference(
    y_true=y_test, y_pred=y_pred, sensitive_features=A_test,
)
```

`MetricFrame` is essentially the per-slice-metrics table from Chapter 1 with fairness-specific summarizers on top. It supports intersectional attributes (pass a `DataFrame` with multiple columns as `sensitive_features`) and any metric compatible with the `(y_true, y_pred)` signature.

**Mitigation primitives.**

- `fairlearn.postprocessing.ThresholdOptimizer` — implements Hardt et al.'s group-specific thresholding to achieve equalized odds or demographic parity.
- `fairlearn.reductions.ExponentiatedGradient` and `GridSearch` — in-processing methods that treat fairness as a constraint on empirical risk minimization.

Fairlearn does not include a pre-processing method by default; it points at community libraries (e.g. AIF360) for that space.

**What Fairlearn does not do.**

- It does not choose the sensitive attribute for you (Chapter 5).
- It does not choose the primary fairness metric for you (this chapter).
- It does not include calibration-within-groups as a first-class primitive — you compute it with the ECE machinery from Chapter 2 per group.

## Aequitas: what it computes and how

[Aequitas](https://dssg.github.io/aequitas/) is the University of Chicago Data Science for Social Good group's fairness audit toolkit. Its emphasis is *audit reports* — machine-readable and human-readable outputs suited to a policy or compliance conversation.

Aequitas takes a labeled prediction table with a sensitive-attribute column and produces:

- **Group metrics.** All the per-group rates in Chapter 5 (selection rate, TPR, FPR, FDR, FOR, PPV, NPV) in a single table.
- **Bias metrics.** Ratios and differences of the group metrics relative to a reference group.
- **Fairness determinations.** Boolean flags for whether each definition is satisfied within specified tolerances (e.g. the four-fifths rule for disparate impact).
- **Audit reports.** HTML/PDF reports scoped to a policy question ("does the model satisfy demographic parity within 10% for all protected attributes?").

A minimal Aequitas flow:

```python
from aequitas.group import Group
from aequitas.bias import Bias
from aequitas.fairness import Fairness

# df must have columns: 'label_value', 'score', and one or more attribute columns.
g = Group()
xtab, _ = g.get_crosstabs(df)   # per-group counts and rates

b = Bias()
bdf = b.get_disparity_predefined_groups(
    xtab, original_df=df,
    ref_groups_dict={"race": "white", "gender": "male"},
    alpha=0.05,
)

f = Fairness()
fdf = f.get_group_value_fairness(bdf)
```

Aequitas leans further into the report-generation and policy-tolerance direction than Fairlearn does; Fairlearn leans further into the mitigation-and-integration-with-sklearn direction. In practice teams use both, or use one for the audit report and one for the mitigation experiments.

## A worked measurement flow

Combining Chapter 5 and this chapter into one end-to-end shape.

1. **Freeze the operating point** from Chapter 4. If the operating point is under discussion, freeze it *for the audit* explicitly and revisit if it changes.
2. **Compute per-group metrics** with `MetricFrame` (Fairlearn) or Aequitas. At minimum: selection rate, TPR, FPR, precision, ECE-per-group.
3. **Compute disparity summaries.** Demographic parity difference and ratio, equalized odds difference, equal opportunity difference, PPV disparity, ECE disparity.
4. **Attach CIs** via bootstrap (mod-101 Chapter 4). Every disparity is a difference of two rates each of which has sampling error; without CIs, small-`n` disparities look larger than they are.
5. **Compute intersectional slices** (age × race, geography × gender) subject to the sparsity flag from Chapter 1.
6. **Write the trade-off memo.** Identify which fairness definition is the primary one, explain why, report the others as evidence, and describe what mitigation would look like if it were chosen. Name the accountable owner.
7. **Rerun after any material change.** Recalibration, threshold change, model retrain, or dataset shift all invalidate the previous audit. The audit is not a one-time artifact.

## Common failure modes specific to the impossibility trade-off

- **Claiming all fairness definitions are satisfied.** Either the base rates are equal (in which case the trade-off is degenerate and you should say so), or someone is not looking at all the definitions. Verify.
- **Ignoring the base rate.** Fairness disparities that trace directly to legitimate base-rate differences are a different problem from disparities the model created; the report should decompose the disparity into "base-rate-attributable" vs. "model-attributable" wherever possible.
- **Choosing the primary metric to minimize the number that looks bad.** Metric selection is a policy decision made before the numbers are visible; making it after is post-hoc rationalization.
- **Post-processing mitigation without disclosure.** Group-specific thresholds require an explicit policy defense; shipping them silently is what motivated a lot of the regulatory attention this space now attracts.
- **Confusing mitigation success with fairness.** A mitigation algorithm that satisfies one definition on the held-out set may fail it in production due to distribution shift, and will generally fail one of the other definitions by the impossibility theorem. Ship the audit alongside the mitigation.

## Summary

The Chouldechova (2017) and Kleinberg–Mullainathan–Raghavan (2017) impossibility results say: when base rates differ across groups, the standard fairness definitions from Chapter 5 cannot all hold simultaneously. That is a theorem, not a modelling failure. The eval engineer's job is to name the trade-off — which definition is primary, why, and what the other definitions look like as evidence — and to run the measurement with the appropriate tooling. Fairlearn's `MetricFrame` and mitigation primitives, and Aequitas's audit reports, are the two most common Python libraries and are complementary in practice. Ship the trade-off memo alongside the numbers; a fairness measurement without a named primary metric and an accountable owner is a spreadsheet, not an audit. Chapter 7 turns from classification to regression and ranking, and picks up the subgroup-breakdown discipline for those other model classes.
