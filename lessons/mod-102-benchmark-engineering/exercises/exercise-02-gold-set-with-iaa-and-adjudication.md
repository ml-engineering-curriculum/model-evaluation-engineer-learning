# exercise-02: Gold Set With IAA and Adjudication

**Estimated effort:** 4 hours

## Objective

Run a small, real annotation pilot end-to-end: write the instructions, collect three-annotator labels on 50 items, compute chance-corrected agreement, adjudicate the residual, and ship a gold set plus an IAA report. The output is a small annotated dataset (~50 items) with a written adjudication log and an IAA memo, not a production benchmark.

The point is to feel where the labelling pipeline breaks: which instructions are ambiguous, which items reveal a construct-definition gap, and how expensive a well-run adjudication actually is.

## Prerequisites

- Chapters 03 and 04 of this module.
- Python 3.11+ with `scikit-learn` (for `cohen_kappa_score`), `statsmodels` (for `fleiss_kappa`), and optionally the `krippendorff` package for α.
- Three collaborators to annotate — or, for solo work, three prompt-engineered LLM annotators used as stand-ins with the caveat below.

If you are running this solo with LLM annotators, that is acceptable for the exercise if and only if you (a) run at least three distinct prompt/model configurations (not three calls to the same config) and (b) explicitly discuss in your writeup how the finding generalizes — or fails to — to human annotators. Use the LLM-annotator variant as a scaffold for later real-annotator work; do not treat the results as substitutable for human IAA in a real benchmark.

## Task choices (pick one)

Pick a construct that is contentious enough to produce disagreement but bounded enough to complete in the time budget. Suggested options:

- **Support-ticket triage.** Classify 50 public support tickets into `{billing, technical, abuse, other}`. Pull items from a public support-ticket dataset (e.g. `bitext/bitext-customer-support-llm-chatbot-training-dataset` on Hugging Face, license-permitting) or synthesize a small set from public product forums with the Chapter 1 provenance record attached.
- **Answer-quality Likert.** For 50 prompt/response pairs from a public open-ended QA source (e.g. LMSYS's chat data with an appropriate license, or synthetic pairs you generate), assign a 1–5 Likert on "answers the user's question."
- **Refusal-appropriateness.** For 50 prompt/response pairs where a chat model refused, label whether the refusal was `appropriate`, `overzealous`, or `wrong-reason`. Draw from a public refusal-audit dataset or curate from public model outputs with source provenance.
- **Toxicity severity.** For 50 short public snippets from a labelled toxicity dataset (Civil Comments, Jigsaw), re-label under a *new* rubric you author, and compare against the released labels as a separate calibration exercise. Use the dataset per its license terms.

Whatever you pick, record the source and license in a mini-dossier alongside the exercise output (Chapter 1 discipline).

## Requirements

### Part A — write the instructions

Author `INSTRUCTIONS_v1.md` following Chapter 3's structure:

- Purpose paragraph with construct definition.
- Label set with per-label glosses.
- Decision rules or judgment scaffolding.
- 2–5 examples per label, including at least one boundary and one near-miss per label.
- Explicit non-goals.
- Escalation rule for when the annotator is unsure.
- Metadata to record per annotation (confidence 1–3, time-taken, flag-for-review).

Target length: 3–8 pages. Version it (`v1.0.0`) and check into the exercise repo.

### Part B — run the pilot

Collect three-annotator labels on your 50 items.

- Each annotator labels every item independently. No collaboration mid-pilot.
- Record labels in a per-annotator CSV or JSONL keyed by `item_id`.
- Include the confidence and time-taken metadata fields.
- Log the instruction version each annotator saw.

Store the raw annotations under `annotations/v1/`.

### Part C — compute IAA

Write `iaa.py` (or a notebook) that produces:

1. **Pairwise Cohen's κ** for each pair of annotators (three pairs for three annotators). Report point estimate and a bootstrap 95% CI over items (mod-101 Chapter 4 pattern).
2. **Fleiss' κ** across all three annotators. Report point + bootstrap CI.
3. **Krippendorff's α** across all three annotators. Report point + bootstrap CI.
4. **Weighted κ** if your labels are ordinal (Likert, severity). Use quadratic weights and state the choice.
5. **Raw agreement fraction** (unanimous / two-of-three / all-differ). Report the three counts.
6. **Confusion matrices** for each annotator pair, with per-cell counts.

State the IAA target you set for this exercise (e.g. κ ≥ 0.70 for a judgment task) *before* you compute κ, in the writeup.

### Part D — adjudicate the disagreements

For every item where annotators disagreed (at least one disagreement), produce an adjudicated gold label with a written rationale using the schema from Chapter 4:

```yaml
- item_id: item-a4b1c
  annotator_labels:
    ann-01: helpful
    ann-02: partially-helpful
    ann-03: helpful
  adjudicator: yourself-or-name
  gold_label: helpful
  rationale: >
    Response answers the primary question; the follow-up gap noted
    by ann-02 is stylistic, not substantive.
  instructions_version: 1.0.0
```

Store adjudications in `adjudication_log.yaml`. For items where all three annotators agreed, no adjudication is needed; the gold label is the shared label.

### Part E — write the IAA + adjudication report

Produce `IAA_REPORT.md` (target length: 800–1500 words) containing:

1. Task summary and construct statement.
2. Annotator description (who they are, expertise, and — for LLM annotators — the model/prompt configurations used).
3. All IAA numbers from Part C with 95% CIs.
4. Verdict: did the pilot meet the pre-stated IAA target? Yes/No + one paragraph.
5. Disagreement analysis: categorize the disagreements into instruction-ambiguity, item-ambiguity, annotator-error, or degenerate-item (Chapter 4 taxonomy). Report counts.
6. **Instructions v1.1 changelog.** Concrete rewrites to the instructions that the adjudication log motivates. At least three specific changes with pointers to the driving items.
7. Recommendations: whether to run a second pilot, whether the construct itself needs redefinition, and what the expected IAA lift would be from the v1.1 rewrite.

### Part F — ship the gold set

Produce `gold.jsonl` with one row per item:

```json
{
  "item_id": "item-...",
  "input": "...",
  "reference": "gold_label_value",
  "metadata": {
    "num_agreeing_annotators": 3,
    "adjudicated": false,
    "instructions_version": "1.0.0"
  }
}
```

`num_agreeing_annotators` records how many annotators picked the gold label in the raw pass (before adjudication). `adjudicated` is `true` for items resolved by the adjudicator.

## Starter guidance

- Do not annotate the same items yourself as the instruction author. If you must, note it explicitly and expect suspiciously high agreement between "you" and the ideal annotator; use the other two annotators' κ against each other as the real signal.
- Stratify your 50 items so at least one item per label class appears. Random draws at n=50 can leave a rare class with zero items and undefined κ.
- If your first pass produces κ < 0.5, that is a signal to stop and re-examine the instructions rather than adjudicate through it. Discuss in the report.
- For LLM annotators, use different providers or clearly different prompt structures. Three calls to the same prompt on the same model at temperature 0 are one annotator, not three, and κ will be artificially high.
- The adjudication rationale is not optional. An adjudication log without rationales is not useful for the v1.1 instructions rewrite.
- Report bootstrap CIs by resampling items (each item = one bootstrap unit) with `B ≥ 1000`.

## Acceptance criteria

Your submission is acceptable if a reviewer can answer "yes" to every item below:

- `INSTRUCTIONS_v1.md` contains every structural element from Chapter 3 (purpose, label set, decision rules, examples-per-label, non-goals, escalation, metadata).
- `annotations/v1/` contains three independent annotator files with 50 items each.
- `iaa.py` produces every IAA metric in Part C with a bootstrap 95% CI. Numbers are reproducible from a documented seed.
- The IAA target was stated *before* computing κ, not chosen to match the result.
- `adjudication_log.yaml` covers every disagreement item, with a rationale for each. No adjudications have empty rationales.
- `gold.jsonl` has one row per item with the required fields, and adjudicated rows are flagged.
- `IAA_REPORT.md` categorizes disagreements per Chapter 4's taxonomy and proposes a concrete v1.1 instruction changelog motivated by specific items in the log.

## Stretch goals

- **Run a second pilot on a fresh 50 items using `INSTRUCTIONS_v1.1.md`** and compare the IAA numbers. Show whether the rewrite closed the specific gaps you targeted, and what residual disagreement remains.
- **Compare a decision-rule labelling** of the same 50 items (write a rubric with an unambiguous rule tree) against your judgment labelling and quantify the trade-off in κ vs. label expressiveness.
- **Simulate gold rotation.** Wait two weeks (or use a different annotator pool), re-annotate 10 of the gold items blind, and compute κ against the previous gold. Interpret drift.
- **Compute per-label F1** treating one annotator as the "prediction" and the adjudicated gold as the "truth." Which labels have the lowest per-annotator F1? Cross-reference with the disagreement log.
- **Weight the κ computation.** For an ordinal task, compute both linear-weighted and quadratic-weighted κ and discuss which better matches the cost of a mis-ordering in your downstream use.
