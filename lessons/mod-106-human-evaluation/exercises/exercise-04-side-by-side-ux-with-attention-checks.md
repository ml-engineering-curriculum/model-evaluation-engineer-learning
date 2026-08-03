# exercise-04: Side-by-Side UX with Attention Checks and Bot Detection

**Estimated effort:** 2 hours

## Objective

Build a minimal but complete side-by-side pairwise annotation UI with the four load-bearing UX decisions from Chapter 6 baked in — per-item randomized presentation, ternary verdict, blinded system identity, no post-submit revision. Layer in three attention-check patterns and a simple bot-detection heuristic (time-per-item lower bound). Run a small self-study (10–20 items with 2–3 annotators, including a deliberate "inattentive annotator" run) to demonstrate the QC primitives fire correctly.

The deliverable is not a production tool. It is a working artifact you can point at during a design review to say "this is what the four UX decisions look like implemented" and demonstrate that the attention checks and bot-detection alerts actually detect what they claim to.

## Prerequisites

- mod-106 Chapter 6 in full.
- One of the following stacks (pick whichever you're fastest in):
  - Streamlit (Python; recommended for the exercise — 1-file UI, easy to run locally).
  - A simple Flask / FastAPI + minimal HTML page.
  - A Jupyter notebook with `ipywidgets` (works but the "session" model is awkward).
- 20–40 candidate prompt / response-A / response-B triples. Sources: any pairwise-preference dataset (Chatbot Arena, alpaca-eval), model outputs you've generated locally, or synthetic triples you author. The content matters less than that the responses are visibly different in quality.

## Requirements

### Part A — the UI

Ship `app.py` (or the equivalent for your chosen stack) implementing:

1. **Item queue.** Each session serves N items sequentially. The queue is per-annotator and persistent across page refreshes (a small SQLite file or JSON blob is fine).
2. **Per-item randomization.** For each (annotator, item) pair, randomly assign each system to the "left" or "right" slot. Log the assignment. The annotator sees "Response A" and "Response B" but which underlying system is A vs. B varies per item.
3. **Ternary verdict.** Buttons: "A is better", "Tie", "B is better". Exactly one is selectable per item.
4. **Blinded system identity.** The UI never shows the underlying system names (`gpt-4o`, `claude-opus`, `llama-3-8b`, etc.) — only "Response A" and "Response B". If you want a post-vote reveal, that's fine, but it must come after the vote is submitted.
5. **No post-submit revision.** Once "Submit" is clicked, the item is locked. No back button changes the label.
6. **Time-per-item logging.** Record the time between the item first appearing and the vote being submitted.
7. **Optional required note field.** A free-text field for the annotator's one-sentence reason. Optional means: it should be configurable, and the exercise expects you to enable it for at least one run so bot detection can use response entropy.

### Part B — three attention-check patterns

Ship `attention_checks.py` implementing at least three attention-check items and inject them into the queue at ~10% rate:

1. **Explicit instruction check.** A pairwise item whose prompt or one of the responses contains an instruction the annotator must follow (e.g., "for quality assurance, select 'Tie' on this item"). Design so the correct behavior is unambiguous under both a naive-attention reading and the guideline.
2. **Trivial correctness check.** A pairwise item where one response is obviously and dramatically better than the other (the other is gibberish, empty, or a copy of the prompt).
3. **Duplicate consistency check.** The same underlying item served twice in the session (with different item IDs, spaced at least 5 items apart). Same content, potentially swapped left/right. An annotator whose two labels on the same content disagree fails this check.

For each attention check, define the expected behavior and log per-annotator pass/fail. Do *not* mark attention-check items as such in the UI; from the annotator's perspective, they are indistinguishable from real items.

### Part C — bot-detection heuristic

Ship `bot_detection.py` implementing at least one bot-detection heuristic. The simplest: **time-per-item lower bound**. For each item, the "read + judge" time should be at least some minimum floor (say, 5 seconds for short items, 10 for long). Compute per-annotator fraction of items completed below the floor; flag annotators with > 25% below-floor items.

Optional stretch: if you enable the required note field in Part A, add a simple response-entropy heuristic — compute the pairwise cosine similarity (or even simpler, character overlap) between an annotator's notes across items and flag annotators whose average pairwise note similarity is unusually high.

### Part D — the self-study

Run the following three sessions:

1. **Attentive annotator (yourself, honest labels).** Complete 15+ items with real judgments. Take your time on each. Expected: all attention checks pass; bot-detection heuristic does not fire.
2. **Inattentive annotator (yourself, deliberately clicking through).** Complete another 15+ items without reading them carefully — click through as fast as possible, alternating "A", "Tie", "B" arbitrarily. Expected: attention checks fail; time-per-item bot-detection heuristic fires.
3. **Second real annotator.** A teammate, friend, or mentor completes 10+ items following the same instructions as session 1. Real judgments. Expected: pattern similar to session 1; attention checks pass.

Log every session with per-item labels, times, and attention-check outcomes.

### Part E — the report

Ship `REPORT.md` (1–2 pages) covering:

1. **UX design summary.** How each of the four load-bearing decisions is implemented. Screenshot or annotated code snippet for at least the randomization logic.
2. **Attention checks.** The three patterns you implemented, with an example of each and the pass/fail logic.
3. **Bot detection.** The heuristic, the threshold, and how you calibrated the threshold (baseline noise on the attentive session).
4. **Self-study results.** Per session: pass rate on attention checks, per-annotator time distribution, whether the bot-detection heuristic correctly identified the inattentive session and cleared the attentive session.
5. **Randomization audit.** For each item, the per-annotator left/right assignment. Aggregate: is the underlying-system-A win rate the same as the underlying-system-B win rate when position is controlled for? (On 40 items it won't be statistically significant either way; the point is that the *logging* is there for downstream analysis.)
6. **What would fail with real crowd workers.** Name at least two additional QC failures you'd expect on Prolific / Mercor that this exercise does not exercise. LLM-mediated labelling is the obvious one; others include cross-session collusion, browser extensions that auto-fill, and workers running multiple accounts.

### Part F — bundle

Ship in the exercise directory:

- `app.py` (or equivalent) — the UI
- `attention_checks.py`
- `bot_detection.py`
- `data/items.jsonl` — the pool of pairwise items, including the injected attention-check items with metadata
- `data/sessions/` — per-session log files (labels, times, attention-check outcomes)
- `analysis/report.py` — computes the summary tables in the report from the session logs
- `REPORT.md`
- `run.sh` — one-shot instructions to launch the app locally and generate the report after sessions are complete

## Starter guidance

- **Streamlit is the shortest path to a working demo.** A minimal side-by-side annotation app in Streamlit is under 100 lines. Do not over-engineer this — the exercise is about the *primitives* being correct, not the polish.
- **Log everything with a stable session ID.** Every submitted vote should record: annotator ID, session ID, item ID, underlying-system-A ID, underlying-system-B ID, left-slot-system ID, right-slot-system ID, verdict, verdict-time-seconds, note (if enabled), attention-check-type (if applicable), attention-check-pass (if applicable). Every future analysis is downstream of this log.
- **Do not skip the duplicate consistency check.** It is the subtlest of the three and the most informative on borderline honest / inattentive annotators. Spacing matters — a duplicate 3 items apart is memorable; 8+ items apart is not.
- **Randomization is per (annotator, item), not global.** If you randomize once at queue-build time, annotator A and annotator B see the same assignment for item X, which loses some of the power of randomization for downstream analysis. Fresh random draw per annotator.
- **On your inattentive session, do not deliberately fail attention checks in a way you know they'll be caught.** The point is to simulate a genuinely inattentive annotator (fast clicking, no reading). If you know an item is an attention check and you deliberately click through it to prove the check fires, you're testing the check, not the annotator. Ideally, have your teammate run the inattentive session under instructions — they don't know which items are attention checks.
- **The randomization audit is not a statistical test on 40 items.** It is a *plumbing* check: verify the left/right slot IDs and the underlying-system IDs are both logged and that downstream aggregation can un-randomize. Report the counts; do not report a "position bias" number.

## Acceptance criteria

- The UI implements all four load-bearing decisions (per-item randomization, ternary verdict, blinded system identity, no post-submit revision), with code visibly implementing each.
- Three attention-check patterns (explicit instruction, trivial correctness, duplicate consistency) are implemented, injected at ~10% rate, indistinguishable from real items in the UI.
- Bot-detection heuristic (at minimum, time-per-item lower bound) is implemented and its threshold is calibrated against baseline noise, not just picked arbitrarily.
- Three self-study sessions are logged (attentive-self, inattentive-self, second-real-annotator) and the report shows the QC primitives correctly discriminate them.
- The session log carries every field needed for downstream un-randomized aggregation.
- The report includes the randomization audit, the per-session QC summary, and named limitations that this local self-study does not exercise.
- The bundle is reproducible: `bash run.sh` starts the app, and after sessions complete, produces the report.

## Stretch goals

- **Note-entropy bot detection.** Enable the required note field; compute pairwise character-3-gram overlap across an annotator's notes; flag high-overlap sessions. Verify it fires when you (or an accomplice) copy-paste the same note across items.
- **Item-difficulty annotation.** Track per-item annotator agreement across your three sessions. Items where all three annotators agreed are "easy"; items with unanimous disagreement (each picked a different verdict) are "hard." Correlate difficulty with per-item response length. This is a mini version of the Chapter 3 pilot analysis.
- **Reveal-after-vote system identity.** After the annotator submits an item, show which underlying systems produced Response A and B. This is a UX flourish that Chatbot Arena uses; it doesn't affect the labels (because the reveal is after submission) but often improves annotator engagement.
- **Compare to a non-randomized baseline.** Add a config flag that disables per-item randomization. Run 10 items with the flag on and 10 with it off; report the per-side win rate. (Small sample; not statistically significant; the exercise is about the *plumbing* to make this experiment possible.)
- **A "who did which item" audit page.** A reviewer-facing page that shows per-item all annotator votes, all attention-check outcomes, and adjudication status. This is the operational view that any real deployment would need.
