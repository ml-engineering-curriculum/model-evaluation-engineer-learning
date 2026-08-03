# exercise-03: Adjudication Procedure and Gold-Set Rotation

**Estimated effort:** 3 hours

## Objective

Design a written adjudication procedure, implement a simulated gold-set injection and rotation system, and demonstrate that the system detects annotator drift, guideline drift, and gold-item staleness on synthetic data with each failure mode injected on purpose. The deliverable is a small operational module (`gold_set.py`) plus a report that a downstream owner could implement against, with the monitoring dashboard specced explicitly.

You will not run this against a real annotator pool for a homework exercise — instead, you'll simulate annotators with configurable behaviors (an accuracy parameter, a drift schedule, a memorization rate) and show that the system's alerts fire when they should.

## Prerequisites

- mod-106 Chapter 5 in full; Chapter 4 for the agreement math.
- Python with `numpy`, `pandas`, `matplotlib`.
- Optional: `sqlite3` or `duckdb` for the label / gold store; a flat CSV / JSONL store is also acceptable.

## Requirements

### Part A — the adjudication procedure

Ship `ADJUDICATION.md`, a 1-page written procedure that a real project could adopt as-is. It must cover:

1. **Who adjudicates.** Name the roles (third annotator, expert / project lead, discussion between the two original annotators) and the routing rules that determine which mechanism handles which item.
2. **When adjudication triggers.** At minimum: initial disagreement, low-confidence flag, escalation flag, wide-gap disagreement on ordinal scales.
3. **The record.** The exact fields your adjudicated-label row carries: `id`, `annotator_a_label`, `annotator_b_label`, `adjudicated_label`, `adjudicator_id`, `adjudication_method`, `adjudication_reason` (mandatory, prose).
4. **The reason field's expected form.** One sentence, referencing the item and the applicable guideline rule. Include a positive and a negative example.
5. **Escalation ladder.** What happens when the third annotator also disagrees, or when the disagreement crosses a guideline-ambiguity threshold that requires guideline revision rather than per-item adjudication.
6. **Metrics reported per week.** Adjudication rate overall, per slice, per annotator pair. What thresholds trigger a guideline-review meeting.

### Part B — the gold-set implementation

Ship `gold_set.py` with the following API:

```python
class GoldSet:
    def __init__(
        self,
        items: list[dict],  # each has 'id', 'input', 'correct_label',
                            #  'guideline_version', 'confidence' (float 0-1)
        max_impressions_per_annotator: int = 3,
        rotation_fraction_per_period: float = 0.10,
    ):
        ...

    def draw_gold(self, annotator_id: str, n: int) -> list[dict]:
        """Return up to n gold items eligible for this annotator.
        Eligibility rules: item is active AND
        item.impressions_for(annotator_id) < max_impressions_per_annotator.
        Prefer items with the lowest impression count."""

    def record_label(
        self, annotator_id: str, item_id: str, label
    ) -> None:
        """Register that annotator labelled the gold item; increment the
        impression counter."""

    def per_annotator_accuracy(
        self, annotator_id: str, window: int = 50
    ) -> tuple[float, int]:
        """Return (accuracy, n) over the annotator's last `window` gold
        labels. Return (nan, 0) if fewer than 10 labels."""

    def pool_accuracy(
        self, window_days: int = 14
    ) -> tuple[float, int]:
        """Return (accuracy, n) across all annotators' labels in the
        last `window_days` days."""

    def item_accuracy(
        self, item_id: str, window_days: int = 14
    ) -> tuple[float, int]:
        """Return (accuracy, n) on this specific gold item across all
        annotators in the window."""

    def rotate(self, period_now: int) -> tuple[list[str], list[dict]]:
        """Retire items that have hit max impressions across the pool
        AND run the scheduled per-period rotation (retire
        rotation_fraction_per_period of the active pool at random,
        add newly-adjudicated items in their place). Return
        (retired_item_ids, added_items)."""

    def rebump_guideline_version(self, new_version: str) -> None:
        """Mark all items as requiring re-adjudication under the new
        guideline. Items pending re-adjudication are still active for
        injection but their labels do not contribute to accuracy
        metrics until adjudication completes."""
```

The persistent store can be in-memory for the exercise, but write it so a real project could swap in SQLite or Postgres by replacing one class.

### Part C — annotator simulator

Ship `simulate.py` — a small simulator that runs an annotation session with a mix of gold and production items, using a configurable model of annotator behavior:

```python
class Annotator:
    def __init__(
        self,
        annotator_id: str,
        base_accuracy: float,  # e.g., 0.85
        drift_schedule: callable = None,  # accuracy over time
        memorization_rate: float = 0.0,  # 0-1
        random_click_rate: float = 0.0,  # simulates inattention
    ):
        ...

    def label(self, item: dict, current_step: int) -> label:
        """Given a gold item with known correct_label, return a label
        drawn from a distribution that reflects the annotator's current
        accuracy for this item (considering base_accuracy at current_step
        under drift_schedule, plus memorization boost if this annotator
        has seen this item before)."""
```

Then, in `simulate.py`, run four scenarios end-to-end for at least 90 simulated days at some plausible daily volume (say, 200 items / annotator / day, 10% gold):

1. **Baseline.** 5 annotators, all with `base_accuracy=0.85`, no drift, no memorization. Expected: per-annotator accuracy stays flat within noise; no alerts.
2. **Individual drift.** One of the 5 annotators degrades to `base_accuracy=0.65` at day 45. Expected: that annotator's rolling accuracy drops within 1–2 rolling windows and their per-annotator drift alert fires.
3. **Guideline drift.** The gold-item "correct" answers shift for 20% of items at day 60 (simulating a guideline revision the gold set wasn't re-adjudicated against). Expected: pool-wide accuracy drops within a week and the guideline-review alert fires.
4. **Memorization.** All annotators have `memorization_rate=0.15` (each impression of a gold item adds 0.15 to accuracy on that item, cumulative). Without rotation, per-annotator accuracy rises slowly over 90 days. With `max_impressions_per_annotator=3`, it does not.

For each scenario, produce a plot (matplotlib is fine) of per-annotator rolling accuracy over time, with the alert-firing timestamps marked, and a short prose analysis of what the alert stack correctly detected and what (if anything) it missed.

### Part D — the monitoring spec

Ship `MONITORING.md` — a 1-page spec for the operational dashboard a real project would build. Include:

1. **Panels.**
   - Per-annotator rolling gold accuracy over the last 30 days (line plot, one line per active annotator).
   - Pool-wide gold accuracy over the last 90 days.
   - Adjudication rate over the last 30 days, overall + per slice.
   - Distribution of per-item gold accuracy in the last 14 days (histogram).
2. **Alerts.**
   - Per-annotator accuracy drops > 10 percentage points below their 90-day baseline → flag for review.
   - Pool-wide accuracy drops > 5 percentage points over a 14-day window → flag for guideline review.
   - Individual gold item drops below 70% for two consecutive weeks → flag for re-adjudication.
   - Adjudication rate exceeds 40% overall → flag for guideline review.
3. **Owner.** Who owns each alert type and what the response is.
4. **Data model.** The tables / topics feeding the dashboard. Sketch is enough; you don't need to build the SQL.

### Part E — the report

Ship `REPORT.md` (1–2 pages) covering:

1. **The adjudication procedure summary.**
2. **The gold-set design choices.** Impression cap, rotation fraction, pool size relative to per-annotator window. Justify each.
3. **The simulation results.** Four scenarios, four plots, prose analysis of each.
4. **False-positive risk.** For each alert, an estimate of how often it fires when nothing is actually wrong (e.g., in the baseline scenario, how often the per-annotator drift alert fires spuriously). If any alert has > 5% false-positive rate at your chosen threshold, discuss whether to raise the threshold.
5. **What the system would miss.** Every monitoring stack has blind spots. Name at least two failure modes this system does not detect (e.g., an annotator who is inattentive on production items but attentive on gold items — the system will not catch them).

### Part F — bundle

Ship in the exercise directory:

- `ADJUDICATION.md`
- `MONITORING.md`
- `gold_set.py`
- `simulate.py`
- `test_gold_set.py` — at least 6 unit tests: eligibility rules, impression counting, rotation, per-annotator accuracy under drift, pool accuracy, item accuracy
- `analysis/plots/` — the four scenario plots
- `REPORT.md`
- `run.sh` — one-shot pipeline that runs the tests and the four simulations

## Starter guidance

- **Do not over-engineer the store.** For the exercise, a couple of pandas DataFrames in memory are fine. The point is the *logic*, not the persistence.
- **Rolling windows: pick either "last N labels" or "last K days" and stick with it.** Mixing both makes the analysis harder to interpret. The Chapter 5 examples use last-N labels for per-annotator (bounds noise per annotator) and last-K-days for pool-wide (bounds noise per calendar period). Adopt those unless you have a specific reason to change.
- **Base your alert thresholds on the baseline scenario's noise.** A 10 pp drop from baseline is meaningful; a 5 pp drop is inside noise for a 50-label rolling window with `base_accuracy=0.85`. Compute the actual standard deviation on your baseline scenario before setting thresholds — thresholds tuned to a specific noise level are what turn a monitoring system from useless to useful.
- **The memorization scenario is where rotation earns its keep.** Run it *without* rotation first (comment out the `rotate` calls) and show accuracy silently climbing. Then enable rotation and show it stays flat. This pair of plots is the strongest evidence that the rotation policy is doing something.
- **Do not treat the adjudication procedure as an afterthought.** The 1-page ADJUDICATION.md is a real deliverable. A downstream owner should be able to hand it to a new team lead and have them run the process without ambiguity. If your document reads like a scaffold, tighten it until it doesn't.

## Acceptance criteria

- `ADJUDICATION.md` covers all six required items and includes a positive and a negative example of the reason field.
- `gold_set.py` implements the full API; `test_gold_set.py` has at least 6 unit tests all passing.
- `simulate.py` runs all four scenarios end-to-end and produces the four plots.
- The report includes plots, prose analysis of each scenario, false-positive-rate discussion, and at least two named blind spots.
- `MONITORING.md` covers panels, alerts (with numeric thresholds justified against baseline noise), owners, and data-model sketch.
- The bundle is reproducible: `bash run.sh` runs tests, runs the four simulations, and produces the report artifacts.

## Stretch goals

- **Per-slice adjudication rate as an alert.** Extend the monitoring so an adjudication rate on any single slice > 40% (even if the overall rate is < 40%) fires an alert. Add a scenario to the simulation where one slice's guideline is systematically ambiguous and verify the slice-level alert fires while the overall alert does not.
- **Adjudicator quality.** Extend the store to record which adjudicator handled each item. Compute per-adjudicator "reversal rate" (how often another adjudicator, on re-review, changes the label). Add an alert for adjudicators whose reversal rate exceeds the pool median by > 10 percentage points.
- **Guideline-version-linked gold accuracy.** Split the pool accuracy metric by guideline version — under `guidelines_v1.4`, this pool of golds is at 88%; under `v1.5`, 82%. Use this to spot guideline changes that quietly cost accuracy.
- **Cost overlay.** Add a cost model: adjudication cost per item + review-queue cost per alert + guideline-revision cost. Estimate the total operational cost of running the system for a year at your chosen thresholds. Discuss which knobs (impression cap, rotation fraction, alert threshold) have the largest cost sensitivity.
- **A/B on rotation policy.** Run the memorization scenario at three different `rotation_fraction_per_period` values (5%, 10%, 20%) and one impression-cap value (2, 3, 5). Produce a 3x3 heatmap of "average annotator accuracy at day 90." The right operating point is where accuracy stays flat at the lowest turnover cost.
