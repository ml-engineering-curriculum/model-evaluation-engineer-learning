# Sequential Testing and Confidence Sequences for Continuous Monitoring

The three altitudes up to this point — offline regression, shadow, A/B — all have one property in common: they end. The regression suite runs in CI on a candidate build and stops. The shadow runs for a week and stops. The A/B runs to a pre-committed sample size and stops. In all three, the sample size is fixed in advance and the statistical guarantees follow from Neyman–Pearson-style fixed-`n` testing.

Production monitoring is different. It never ends. The eval runs continuously against a stream of live data and must be able to raise alerts *early* when quality drops, when safety metrics regress, when the judge starts behaving oddly. Naïvely applying fixed-`n` tests in this always-on setting inflates the false-alarm rate catastrophically — the "look 100 times a day" pattern will fire a `p < 0.05` alert several times a day even when nothing is wrong.

This chapter is about the statistical tools that let you monitor continuously without either (a) waiting weeks between reads or (b) drowning in false alarms. The two workhorses are **group-sequential tests with alpha spending** (Lan & DeMets 1983; O'Brien & Fleming 1979) and **confidence sequences** based on martingale concentration inequalities (Howard, Ramdas, McAuliffe, Sekhon 2021; Waudby-Smith & Ramdas 2020). Confidence sequences are the modern reference for open-ended monitoring; group-sequential tests are the reference for A/B experiments that permit interim looks. Both fit under the umbrella of **sequential inference**.

## The problem: naïve peeking inflates the false-positive rate

Suppose you monitor a metric — say, refusal rate on production traffic — and every hour you compute a fresh 95% confidence interval on the current window's data, and you alert whenever the CI excludes the pre-drop baseline. If the underlying distribution is stationary (no real drop), each individual look has a 5% probability of a false alarm. Across 24 hourly looks in a day, the probability that at least one look raises a false alarm is well above 5% — approaching 70% under mildly optimistic independence assumptions, and higher under realistic correlation.

The classical mistake is treating each look as an isolated fixed-`n` test. The correct discipline treats the *sequence of looks* as a joint testing problem and controls the *sequence-wide* false-positive rate. There are two families of solutions.

- **Group-sequential tests.** Pre-declare a small number of interim looks (say, 3 or 5), pre-allocate an alpha spend at each look, adjust critical values so the total type-I error across all looks equals your target α. Reference: Pocock 1977, O'Brien & Fleming 1979, Lan & DeMets 1983 for the alpha-spending framework. Fits naturally to A/B experiments with planned peeks.
- **Confidence sequences.** A sequence of confidence intervals `CI_t` for `t = 1, 2, 3, …` such that the probability that *any* CI in the sequence fails to cover the true parameter is at most α — for *any* stopping rule, adaptively chosen. Reference: Howard, Ramdas, McAuliffe, Sekhon 2021; the "always-valid p-value" work of Johari, Koomen, Pekelis, Walsh 2017. Fits naturally to continuous monitoring where the number of looks is unbounded and the stopping rule is data-dependent.

Continuous production monitoring almost always calls for confidence sequences. This chapter covers both but weighs toward the confidence-sequence side because it is the harder concept and the one this altitude specifically requires.

## Group-sequential tests in one page

For completeness before we move on. In a group-sequential test:

- You pre-declare the schedule of interim analyses — e.g., at 25%, 50%, 75%, and 100% of the maximum sample size.
- You pre-declare an alpha-spending function `α(t)` — a monotonically increasing function that determines how much of your total α is "spent" by look `t`. O'Brien–Fleming spends very little early (very hard to reject at first look) and most of it at the final look; Pocock spends alpha uniformly across looks (easier to reject early, harder at the end); Lan–DeMets is the flexible framework these fit inside.
- At each look, you test with a critical value derived from the remaining α to spend. If you reject at any look, you stop and declare a positive; if you never reject, the experiment ends at the final look with the standard conclusion.

Group-sequential tests are the discipline for A/B experiments that want to peek without breaking α. They require you to commit to the schedule and the alpha-spending function in advance — after which you may stop early and the stop is statistically defensible. See Kohavi et al. 2020 §17 for the practical A/B application; Jennison & Turnbull 2000 is the textbook reference.

The limitation: the number of interim looks and the maximum `n` must be pre-committed. This does not fit continuous production monitoring, where you might look every minute for a year and never plan to stop.

## Confidence sequences: the always-valid interval

A **confidence sequence** at level `(1-α)` for a parameter `θ` is a sequence of random intervals `(L_t, U_t)_{t≥1}` such that

`P( ∃ t ≥ 1 : θ ∉ (L_t, U_t) ) ≤ α`.

Compare to a fixed-`n` CI: `P( θ ∉ (L, U) ) ≤ α` at the single value of `n` chosen in advance. The confidence-sequence guarantee is stronger: no matter how many looks you take, no matter what stopping rule you use — including "stop the first time the CI excludes zero" — the probability that any interval in the sequence excludes the true θ is bounded.

The equivalent language in the "always-valid p-value" formulation (Johari, Pekelis, Walsh 2017): a sequence of p-values `p_t` such that `P( ∃ t : p_t ≤ α ) ≤ α`.

The consequence for monitoring: you can look at a confidence sequence at every point in time, and raise an alert whenever the CI excludes the baseline, and the false-alarm rate is bounded by α *even in the worst case over all stopping rules the on-call could ever use.* This is exactly the guarantee that always-on monitoring needs.

### How confidence sequences are constructed

The mathematical machinery is martingale concentration. The intuition, without the proofs:

- Every observation `X_t` is combined with previous observations into a running estimator (say, the sample mean of a bounded random variable).
- A concentration inequality gives an upper bound on how far the running estimator can deviate from its true expectation, with a bound that depends on `t` and is designed to hold *uniformly in `t`*.
- The interval is `estimator ± bound(t)`. The bound shrinks with `t`, but slightly more slowly than the fixed-`n` bound — a `sqrt(log(log(t)) / t)` rate instead of `sqrt(1/t)`, per the law of the iterated logarithm.

The practical upshot: **a confidence sequence is a "wider fixed-`n` CI" — typically 20–50% wider than the equivalent fixed-`n` CI at the same `n`.** That is the price you pay for peeking freely. In exchange, you can peek any time, any number of times, and adaptively stop based on the data.

### Two constructions worth knowing

**Empirical Bernstein confidence sequence for bounded means.** For bounded observations `X_t ∈ [a, b]` with true mean `μ`, Howard et al. 2021 give a construction that computes a running mean `x̄_t`, a running variance estimate `V_t`, and forms an interval whose half-width shrinks like `sqrt(V_t * log(log(t)) / t)`. This is the go-to for monitoring bounded metrics (proportions, judge scores in [0, 5]).

**Betting-based confidence sequences for bounded means.** Waudby-Smith & Ramdas 2020 develop a different family based on "betting" against a null hypothesis. In practice these often give tighter intervals than the empirical-Bernstein construction, especially when the variance is small compared to the range. The `betting_cs` implementation in the `confseq` Python package is a good default.

For proportions specifically (the case that matters most for refusal rate, over-refusal rate, safety flags), both constructions specialize to intervals that can be computed in a few lines. A sketch of the empirical-Bernstein version (real implementations should use a battle-tested library, not this sketch):

```python
import math

def eb_confidence_sequence(bounded_observations, alpha=0.05, a=0.0, b=1.0):
    # Returns a running sequence of (mean, lower, upper) at each t.
    n = 0
    running_sum = 0.0
    running_sum_sq = 0.0
    out = []
    for x in bounded_observations:
        n += 1
        running_sum += x
        running_sum_sq += x * x
        mean_t = running_sum / n
        var_t = max(1e-12, (running_sum_sq - n * mean_t * mean_t) / max(1, n - 1))
        # A schoolbook empirical-Bernstein radius that respects law-of-iterated-log spacing.
        loglog_t = math.log(max(math.e, math.log(max(math.e, float(n))))) if n > 2 else 1.0
        radius = (b - a) * math.sqrt(2 * var_t * loglog_t / n) + (b - a) * 7 * loglog_t / (3 * n)
        # For α control, pick the constant multiplier from the reference (typically ~1.7*sqrt(log(1/α))).
        radius *= math.sqrt(math.log(2.0 / alpha))
        out.append((mean_t, mean_t - radius, mean_t + radius))
    return out
```

Two disclaimers on the sketch above: (a) the constants and precise inequality are simplified; use `confseq` (the reference Python package by the Howard–Ramdas group), `pymc`, or an internal implementation validated against their reference; (b) the sketch is O(t) per observation, which is fine, but the incremental variance formula is numerically fragile — Welford's algorithm is the correct incremental formulation.

## The monitoring plumbing

The math is one thing; wiring it into a monitoring system is the other. A production monitor built on confidence sequences has four pieces:

1. **Ingest.** A stream of `(timestamp, value)` observations per metric per slice. For a proportion metric, `value ∈ {0, 1}`; for a bounded judge score, `value ∈ [0, max]`.
2. **State.** Per (metric, slice) monitor, incremental sufficient statistics — count, sum, sum-of-squares — updated on each observation.
3. **Interval computation.** On each observation (or on a downsampled cadence — every 100 or 1000 observations to keep the alert channel volume manageable), compute the current confidence-sequence lower and upper bound.
4. **Alerting rule.** Compare the current interval to a pre-committed alerting threshold. Two common shapes:
   - **Absolute threshold.** "Alert if the upper bound of the refusal-rate CI drops below 0.98." Anchored to a committed number (as with safety-gate design in Chapter 2).
   - **Baseline-relative threshold.** "Alert if the upper bound of the helpfulness-mean CI is below `baseline_mean - 0.05`." Baseline is a slow-moving reference (rolling 28-day mean, or the value at the last release ramp).

The confidence sequence's guarantee is that the *false-alarm rate over the lifetime of the monitor* is bounded by α. This lets you set α to a value that reflects the on-call cost budget for that alert — a safety metric might get α = 0.001 (one false alarm per 1000 monitor-lifetimes in expectation), a quality metric α = 0.01, and an experimental drift signal α = 0.05.

### Choosing the alpha budget

The confidence-sequence α controls the sequence-wide false-alarm rate, but individual monitors do not know about each other. If you run 200 monitors and each has α = 0.05, the union of false-alarm events across the fleet can approach 200 * 0.05 = 10 false alarms per lifetime — a lot of paging.

Two disciplines to keep this manageable:

- **Bonferroni across monitors, or FDR across monitors.** Mod-101 Chapter 4's Benjamini–Hochberg discipline applies here: the "family of tests" is now the collection of monitors, and the false-discovery rate across the family is what the on-call cares about. FDR-controlled alerting sets tighter individual thresholds for larger fleets.
- **Tiered severity with different α budgets.** Not every metric gets the same alerting threshold. Safety metrics get α = 0.001; latency SLA metrics α = 0.01; long-tail drift signals α = 0.05. Coupled with tiered runbooks (P1 page for safety, P3 ticket for drift), the alerting volume is manageable and the signal-to-noise stays high.

### Judge-drift monitoring: a special case

A special monitor that this altitude enables — and that fixed-`n` methodologies handle poorly — is **judge drift**. The LLM-as-judge that grades your quality metrics is itself a model. When the vendor upgrades it (a silent point-release, a scheduled model deprecation, a routing change), the judge's scores can shift by 1–3 points on the same distribution of inputs, and your quality metric jumps for reasons that have nothing to do with the target model.

The design: maintain a **judge-canary set** — a small fixed set of `(prompt, response)` pairs with human-gold labels — and continuously grade this set with the current judge. A confidence sequence on the judge's agreement rate with the gold labels immediately detects a shift in judge behavior. When the sequence alerts, the correct response is not to re-baseline the quality metrics silently; it is to **pin the judge to the pre-drift version if the vendor permits, or to explicitly re-baseline downstream gates and thresholds against the new judge as a documented event.**

This monitor is the single most useful piece of always-on quality infrastructure in a production LLM eval stack. Without it, a vendor's silent judge upgrade produces a phantom quality metric change that will be attributed to the target model.

## Group-sequential in the A/B setting

For A/B experiments that need the ability to stop early, group-sequential tests are usually a better fit than confidence sequences because the schedule is planned and the fixed maximum `n` is known. A common shape:

- Plan for 4 interim looks: at 25%, 50%, 75%, and 100% of the pre-committed maximum `n`.
- Use O'Brien–Fleming alpha spending — very conservative early, most alpha spent at the final look.
- Compute the O'Brien–Fleming critical value at each look; reject if the standardized test statistic exceeds the critical value. The `rpy2` package can wrap R's `gsDesign` package; native Python options include `statsmodels`'s group-sequential tools.

The properties: the maximum sample size grows by ~10% compared to a fixed-`n` design of the same power, but the expected sample size under a true effect shrinks by 30–50% because the experiment can stop as soon as the effect is clear.

Confidence sequences and group-sequential tests interoperate: a group-sequential test can be run alongside a confidence sequence for exactly the same experiment, and the two give congruent stopping decisions. In practice you usually pick one and stick with it; the choice reflects whether the experiment has a fixed maximum `n` (group-sequential) or is genuinely open-ended (confidence sequence).

## Reporting a confidence-sequence monitor

A monitor's dashboard should show, per (metric, slice):

- The current running mean (or proportion).
- The confidence-sequence lower and upper bounds.
- The alerting threshold (as a horizontal line on the plot).
- The historical trace of the mean and the bounds over the last N days.
- The count of samples in the current window.
- The α budget for the monitor and its family.
- The last alert timestamp and the incident it was tied to (if any).

A minimum text summary of a monitor's state:

```
Monitor: safety.refusal.production.self_harm
Metric family: safety (α = 0.001 per monitor, FDR-controlled across 12 safety monitors)
Baseline: 0.996 (rolling 28-day mean)
Threshold: alert if upper bound < 0.990

Current state:
  n (last 7 days rolling):       48,203
  running proportion:             0.9958
  confidence-sequence lower:      0.9946
  confidence-sequence upper:      0.9968
  status: OK (upper 0.9968 ≥ threshold 0.9900)

Last alert: 2026-06-12 03:14 UTC (INC-4271, resolved 2026-06-12 07:52)
```

The plot version is more informative but the text form fits in a status page and machine-parseable alerting flows.

## The three most common pitfalls

### Pitfall: treating a monitor's alert as a diagnosis

A confidence-sequence alert says "the distribution of this metric has drifted from the baseline; the drift is statistically distinguishable from noise given the α budget." It does *not* say what caused the drift, whether the model is worse or the traffic changed, or whether the change is temporary. The runbook that fires from the alert is a diagnostic workflow, not a rollback trigger. Chapter 6 walks the runbook shape.

### Pitfall: using a confidence sequence for a settled A/B decision

Confidence sequences are conservative by construction — they trade tightness for the peek-any-time guarantee. If you have a fixed-`n` A/B where the sample size is committed and there is no peeking, use a fixed-`n` CI or a group-sequential test; do not use a confidence sequence and complain that the intervals are wider than the ones you get from `scipy.stats.ttest_ind`. The confidence sequence is buying you a different guarantee, and paying for it in interval width.

### Pitfall: forgetting the monitor's alpha ages

A monitor that has been running for a year has already spent some of its α budget on the looks it has taken so far. Confidence sequences are constructed so that the α is bounded across the entire lifetime — including all past looks — so this is not "wrong," but it does mean that a long-lived monitor's confidence intervals get proportionally wider than a freshly-instantiated monitor's. If you inherit a monitor that has been running for a year with a high false-alarm history, it may be more useful to reinitialize it (documented) than to keep spending α against an increasingly-conservative envelope.

## Guidance for the eval author

- **Fixed-`n` for the offline suite, shadow, and A/B; confidence sequences for anything that runs forever.** Pick the tool that matches the setting; do not use a confidence sequence for a batch analysis or a fixed-`n` CI for a rolling monitor.
- **Set α by monitor severity.** Safety monitors get tight α (0.001), quality monitors moderate α (0.01), drift monitors looser α (0.05). Tie the α to the on-call runbook priority.
- **FDR across the monitor fleet.** Independent monitors can jointly inflate the alert volume; apply BH or a family-level Bonferroni to keep the pager sane.
- **Include a judge-canary monitor.** Judge drift is the silent quality-metric mover and is easy to miss without an explicit monitor.
- **The alert is a diagnostic, not a conclusion.** Runbook drives root-cause; the confidence sequence just says "something moved."
- **Reference implementation, not schoolbook math.** Use `confseq` or a validated internal implementation; the schoolbook inequalities have subtle constants and get them wrong the first three times.

## Summary

Continuous production monitoring cannot be built on fixed-`n` tests without either paging on noise or waiting so long between reads that the alerts arrive too late to matter. The two workhorse tools for always-on inference are group-sequential tests with alpha spending (for A/B experiments with planned peeks and a committed maximum sample size) and confidence sequences (for monitors that run forever with unbounded looks and arbitrary stopping rules). Confidence sequences give a peek-any-time, stop-any-time guarantee at a modest widening of the interval — 20–50% wider than the equivalent fixed-`n` CI — and are the reference tool for continuous quality, safety, and drift monitoring. A defensible monitor spends its α budget by severity, controls false-discovery across the monitor fleet, includes a judge-canary as a first-class monitor, and treats an alert as a diagnostic trigger rather than a rollback signal. The next chapter takes the monitors and the metrics from this chapter and wires them into the three widely-adopted LLM observability platforms — Arize Phoenix, Langfuse, and W&B Weave — with an explicit walk of drift and judge-drift alert design.
