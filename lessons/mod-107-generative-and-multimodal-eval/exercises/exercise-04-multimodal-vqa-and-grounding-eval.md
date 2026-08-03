# exercise-04: Multimodal VQA and Grounding Eval

**Estimated effort:** 3 hours

## Objective

Evaluate a vision-language model on a VQA benchmark, add a POPE-style hallucination probe as a grounding check, add a bespoke slice from a modality the primary benchmark does not cover, and produce a *coverage-first* writeup that argues explicitly which visual surfaces the model has and has not been measured on. The deliverable is not a single VQA number; it is a per-slice profile with a documented coverage gap.

## Prerequisites

- mod-107 Chapter 6 (multimodal eval: VQA, alignment, and modality coverage).
- mod-104 Chapter 7 (prompt-format sensitivity).
- Access to a vision-language model. Any of: GPT-4o / GPT-4o-mini via API, Claude with vision via API, Gemini via API, Llama-3.2-Vision, Qwen2.5-VL, InternVL, LLaVA via local serving.
- Python environment with the multimodal-eval harnesses of your choice — `lm-evaluation-harness` supports several VQA tasks; UK AISI Inspect also has multimodal solvers; the LMMs-Eval fork of lm-eval-harness (`EvolvingLMMs-Lab/lmms-eval`) is the most active multimodal-eval harness.

## The benchmarks and probes

You will evaluate on three sources:

1. **A VQA benchmark.** Pick one of: VQAv2 (val split, or the widely-used mini-val subset), OK-VQA, or GQA. Use a well-defined subset — a full VQAv2 val is 200k questions, which is more than a 3-hour exercise can budget for. 1,000–2,000 items is a reasonable target.
2. **POPE** (Li et al. 2023) as a hallucination probe. Use the "adversarial" split (~3,000 items) or a 500-item sample.
3. **A bespoke slice.** Curate 50–100 items from a *modality* the primary benchmark does not cover. Options:
   - **Chart understanding.** Screenshots of matplotlib / seaborn charts you generate, with questions about the data (max value, trend, comparison across bars). Reference answers derivable from the underlying data.
   - **Document layout.** Pages from arXiv PDFs or Wikipedia infoboxes with layout-anchored questions.
   - **UI screenshots.** Screenshots of a public web app with questions about the interface state.
   - **Diagram understanding.** Simple science-diagram images with structural questions.

The bespoke slice should be sourced from *your own product surface* if you have one. Otherwise the chart or document slice is the most instructive.

## Requirements

### Part A — the VQA run

Ship `eval/vqa/run.py`:

1. Loads the chosen benchmark subset (1000–2000 items).
2. For each item, formats the prompt correctly for your model (system prompt + image + question). Log the prompt template.
3. Requests a short answer (VQA answers are typically 1–3 words). Set a `max_tokens` budget appropriately.
4. Extracts the model's answer with a normalizer (lowercase, strip articles, strip punctuation — the standard VQA normalization).
5. Scores against the multi-annotator VQA accuracy metric — `min(#matching_annotators, 3) / 3` when there are 10 human answers; adapt for GQA / OK-VQA reference formats.
6. Aggregates overall accuracy and per-slice accuracy where the benchmark provides slices (question-type for VQAv2, semantic category for GQA).

### Part B — POPE hallucination probe

Ship `eval/pope/run.py`:

1. Loads a POPE split (500-item sample is fine).
2. For each item, asks the model "Is there a `[object]` in this image? Answer yes or no."
3. Parses yes/no from the response.
4. Reports:
   - **Accuracy.**
   - **Precision, recall, F1 on the "no" class** — POPE is designed so that "no" (object not present) is the hallucination-sensitive class. High precision on "no" = the model doesn't spuriously claim absent objects. Recall on "no" = the model doesn't spuriously assert absent-object presence.
   - **Yes-bias rate** — the fraction of "no" ground-truth items the model answered "yes" to. This is the direct object-hallucination measurement.

### Part C — the bespoke slice

Curate `eval/bespoke/items.jsonl` (50–100 items). One JSON object per line:

```
{
  "id": "chart-001",
  "image_path": "images/chart-001.png",
  "question": "What is the maximum value on the y-axis?",
  "reference_answers": ["100", "100.0"],
  "modality": "chart",
  "difficulty": "easy"
}
```

Include a mix:

- Easy items whose answer is directly readable from the image (a labelled axis, a number in a title).
- Medium items requiring one derivation step (comparing two bars, reading an intersection).
- Hard items requiring counting, arithmetic, or multi-step visual reasoning.

Ship `eval/bespoke/run.py` that runs the model against these items and scores with a normalized-answer accuracy (numeric tolerance for numeric answers, exact/normalized match otherwise). For open-ended items where multi-answer matching is not applicable, hand-label pass/fail after the run.

### Part D — the report

Ship `REPORT.md` (≤ 3 pages) covering:

1. **Setup.** Model, provider, image preprocessing (resize, tile), per-image token budget, prompt templates for each of the three eval passes.
2. **Numbers.**
   - VQA overall accuracy + per-slice.
   - POPE accuracy + precision/recall/F1 on "no" + yes-bias rate.
   - Bespoke slice accuracy overall + per-modality-subtype + per-difficulty.
3. **Coverage argument.** Enumerate the visual modalities the three eval passes together cover. For each modality *your product actually encounters* (if you have one) or a plausible representative product, note which ones are covered and which aren't. This is the payoff of the exercise.
4. **Failure examples.** 5 items — from any of the three passes — where the model got it wrong, with a one-line diagnosis (hallucinated object, misread axis, wrong count, wrong reading order, mis-grounded spatial claim).
5. **Prompt-sensitivity check.** Rerun 200 VQA items with the image placed *before* vs. *after* the question in the prompt. Report the accuracy delta. This is the mod-104 Chapter 7 discipline applied to VLMs.
6. **Reproducibility manifest.** Model version, harness version, image preprocessing config, prompt templates, sampling parameters, dataset commit.

### Part E — bundle

- `eval/vqa/run.py`, `eval/vqa/logs.jsonl`
- `eval/pope/run.py`, `eval/pope/logs.jsonl`
- `eval/bespoke/items.jsonl`, `eval/bespoke/images/`, `eval/bespoke/run.py`, `eval/bespoke/logs.jsonl`
- `REPORT.md`
- `README.md` — rerun instructions

## Starter guidance

- **Do not use a bigger VQA slice than 2000 items.** VQA cost scales with per-image tokens; a 200k-item run is neither useful nor tractable at this exercise's budget. A 1000–2000 item VQAv2 slice at a fixed random seed is comparable across models — that is what published mini-VQAv2 reports use.
- **Image preprocessing matters more than you think.** GPT-4o downscales large images to a fixed max side; Anthropic's SDK does its own resize; local `llava.cpp` builds do yet another. Fix the resize/tile strategy explicitly in your adapter and log it. A 5-point score gap on the same benchmark between two "same-model" runs is almost always image-preprocessing.
- **POPE's "yes-bias" is the single most useful multimodal probe I know.** A model at high VQA accuracy but with yes-bias > 0.2 is one that answers confidently regardless of what's actually in the image. This is the specific case where VQA accuracy alone hides catastrophic behaviour. Read the POPE paper's §3 for the design rationale.
- **The bespoke slice is where you learn the coverage argument.** Even 50 items from a modality VQAv2 doesn't touch — charts, screenshots, receipts, whatever — will typically be far below the model's VQAv2 accuracy. That gap *is* the argument that leaderboard VQA numbers do not generalize.
- **Do not judge open-ended answers with the VQA metric.** VQA accuracy is designed for short-answer multi-annotator scoring; on longer answers, use a rubric-graded judge (mod-105) or hand-label. The three passes deliberately keep answers short.
- **Log per-item image size.** Some benchmarks have images that get down-scaled below the size at which any small text is legible. If your bespoke slice includes text-in-image items, verify the preprocessing didn't destroy the signal before blaming the model.
- **Chart items are easy to generate.** `matplotlib` → PNG with a hand-authored question is a low-friction way to source items. The trap: keep the visual clarity high; if your chart is illegible even to you at the resolution the model sees, the eval is measuring OCR, not chart understanding.
- **Compare against a published number if you can.** For VQAv2 mini-val on GPT-4o, there is a published number; for LLaVA, there are published numbers. A > 5 point gap = image preprocessing or prompt template drift; diagnose.

## Acceptance criteria

- VQA run reports overall accuracy plus at least one per-slice breakdown (question-type or category) on 1000+ items.
- POPE run reports accuracy plus precision/recall/F1 on "no" plus yes-bias rate.
- Bespoke slice has 50+ items in at least one modality not covered by VQA, with per-difficulty and (where meaningful) per-subtype breakdowns.
- `REPORT.md` has a coverage argument section that names the modalities covered by the three passes and identifies at least two modalities the suite does not measure.
- The prompt-sensitivity check on 200 VQA items with image position swap is run and the delta reported.
- The reproducibility manifest pins model version, image preprocessing, prompt template, and dataset commit.

## Stretch goals

- **A second model.** Run everything on a second VLM of a different family. Per-slice accuracy comparisons — especially on the bespoke slice — are usually more informative than the aggregate. If one model is at 60% on charts and 90% on natural photos while another is 80% and 82%, the product-fit answer depends on which of your surfaces matters.
- **MMMU or MathVista subset.** Add a 500-item MMMU or MathVista subset as a "reasoning-heavy" pass. Comparing your model's MMMU score to its VQAv2 score is the empirical version of "leaderboard columns don't span vision." A model can be very good on VQAv2 and very bad on MMMU (or vice versa).
- **ChartQA or DocVQA as a second benchmark.** Instead of (or in addition to) the bespoke slice, run ChartQA or DocVQA. Compare accuracy on the public benchmark to your bespoke chart items. The gap tells you whether your bespoke slice is harder or easier than the field-standard one — either answer is informative.
- **CLIPScore as an alignment measurement.** For a caption-generation subtask (ask the model to describe an image), compute CLIPScore between the generated caption and the image. Compare CLIPScore-ranking to human-preference ranking on 50 pairs. This is the concrete demonstration of CLIPScore's failure modes.
- **A grounded-instruction rubric.** Use a judge (with vision) to rubric-grade 100 open-ended "describe what's happening in this image" items. Per-criterion scores on grounding, coverage, and hallucination. This is the mod-105 machinery applied to the multimodal setting.
- **A prompt-template sweep.** Rerun 200 VQA items under 4–5 different prompt templates (system prompts, image-first vs. question-first, "answer concisely" vs. no instruction). Report the range. A range > 5 points is the direct measurement of how much prompt template noise you are shipping under the aggregate.
