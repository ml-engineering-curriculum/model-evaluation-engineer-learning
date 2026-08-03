# Multimodal Eval: VQA, Alignment, and Modality Coverage

Vision-language models — GPT-4V, Claude with vision, Gemini, Llama-3.2-Vision, Qwen-VL, InternVL, LLaVA — accept an image alongside a text prompt. Everything from mod-104 through mod-106 still applies (harness, judge, human) but with an added degree of freedom: the input now includes pixels, and the eval has to interrogate whether the model *saw* the image in the way the task requires. This chapter walks the three canonical measurement instruments — VQA-style short-answer accuracy, image-text alignment scores, and grounding checks for visually-anchored instruction following — and then makes the central argument specific to this family: *modality coverage* is a first-class eval-design question, not something you get for free by picking a popular benchmark.

## The three instruments

### 1. Visual Question Answering (VQA)

**Shape.** Input: `(image, question)`. Output: a short answer (a word, a phrase, sometimes a sentence). Metric: accuracy against a set of human-provided reference answers.

**Canonical benchmarks.**

- **VQAv2** (Goyal et al. 2017). 265k images from COCO with ~5.4 questions per image, 10 human answers per question. Answers are typically one to three words. VQAv2's scoring formula is the *VQA accuracy metric*: for a given predicted answer, `min(number of human answers that match, 3) / 3`, so a prediction agreed on by ≥3 of 10 annotators gets full credit. This tolerates the natural variation in short-answer phrasing (`man` vs. `person`) without either requiring exact match or hand-writing a synonym list per item.
- **OK-VQA** (Marino et al. 2019). ~14k questions requiring external world knowledge beyond what's depicted (e.g. "What year was this stadium built?"). Same short-answer accuracy metric.
- **GQA** (Hudson & Manning 2019). Scene-graph-based questions that stress compositional reasoning ("Are the plates on the left of the wine glass?"). Includes a "distribution" metric that measures answer diversity, on top of raw accuracy.
- **TextVQA** (Singh et al. 2019). Questions that require reading text *in* the image. Distinct capability from natural-scene VQA.
- **MMMU** (Yue et al. 2024). ~11.5k college-course-style questions across 30 subjects (arts, business, science, medicine, humanities, engineering, tech). Multi-image, multi-choice / short-answer. Deliberately designed after early VQA benchmarks saturated; MMMU is far from saturation as of writing.
- **MMBench** (Liu et al. 2023). ~3k multi-choice questions with a circular-eval trick (shuffle answer positions across passes) to detect models that guess based on position.
- **MathVista** (Lu et al. 2024). ~6k math problems that require visual understanding (charts, diagrams, geometry figures).

**What VQA scoring gets right.** The multi-annotator accuracy metric is well-designed for the short-answer setting — it does not require exact match, does not require a hand-written synonym list, and produces a well-calibrated per-item score. On natural-scene questions with short answers it is the correct measurement.

**What it does not tell you.** VQA-style accuracy does not measure open-ended visual reasoning, long-form image description quality, or whether the model is grounding its answer in the image at all (see grounding below).

### 2. Image-text alignment

**Shape.** Given an image and a candidate caption or description, score how well the caption *aligns* with the image.

**Canonical instruments.**

- **CLIPScore** (Hessel et al. 2021). The cosine similarity between the CLIP image embedding and the CLIP text embedding of the caption. Reference-free. Produces a single scalar per (image, caption) pair. Widely used because it does not require ground-truth captions.
- **BLIP / BLIP-2 ITM** (Li et al. 2022). Image-text matching head trained specifically for alignment classification; more accurate than CLIPScore on hard negatives.
- **Reference-based** (BLEU/CIDEr/SPICE against gold captions). Older, still used for image-captioning benchmarks like COCO Captions; the failure mode is the classic n-gram-overlap failure that motivates model-graded scoring in the first place.

**What alignment scoring gets right.** Reference-free — CLIPScore requires only the image and the candidate text — makes it viable for production monitoring of generated captions, alt-text, or vision-grounded chat replies at scale.

**What it does not tell you.** CLIPScore inherits the CLIP model's biases and blind spots. It over-rewards captions that use words CLIP embeddings cluster near the image, and under-rewards technically correct but rare phrasings. It does not detect *hallucinated* objects reliably — a caption that mentions a dog in an image without a dog can still score middlingly if the rest of the caption is well-aligned. The failure modes are well-characterized in the CLIPScore paper itself (Hessel et al. 2021 §4).

### 3. Vision-grounded instruction following

**Shape.** Input: `(image, instruction)` where the instruction refers to something visually depicted. Output: a response that requires actually looking at the image. Metric: task-specific — often a rubric-scored judge on the response's *use* of visual information.

**Why this is separate from VQA.** VQA questions typically have short, verifiable answers. Instruction-following on images includes tasks like "describe what's in the top-right corner," "identify all the safety violations in this construction-site photo," "count the number of red items," "translate the sign in the image." These have longer, less-constrained answers.

**Instruments.**

- **POPE** (Li et al. 2023) — Polling-based Object Probing Evaluation. Reference-based hallucination test: for each image, ask "Is there a [X] in this image?" over adversarially chosen X. Ground-truth is derived from segmentation annotations. Reports accuracy, precision, recall on hallucinated-object detection.
- **HallusionBench** (Guan et al. 2024). Adversarial vision-language hallucination benchmark, mixed image types and question shapes.
- **MMHal-Bench** (Sun et al. 2023). Hallucination measurement in image-QA responses.
- **Grounding datasets.** RefCOCO / RefCOCO+ / RefCOCOg for referring-expression comprehension — the model must return bounding box coordinates or click positions. If your product needs pointing/segmenting behaviour, this is the correct benchmark family.
- **Judge-graded rubric evals.** For open-ended tasks (image description, visual analysis), a rubric-scored judge — usually the same LLM-as-judge machinery as mod-105, adapted to take the image as input — is the practical answer. Grounding, correctness, and coverage are the typical rubric criteria.

**What grounding scoring gets right.** POPE-style probes cheaply expose object-hallucination behaviour that would otherwise hide behind fluent output. Rubric-scored grounding evals let you evaluate open-ended tasks that VQA short-answer accuracy cannot.

**What it does not tell you.** Object-hallucination probes measure one kind of grounding (presence/absence of named entities). They do not measure spatial relation errors ("the cup is left of the plate" when it is right of it), counting errors ("three people" when there are four), or fine-grained attribute errors ("red car" when it is orange). Different products need different grounding checks.

## The four objects, translated for multimodal

- **Model adapter.** Now accepts `(text, image_or_images)` inputs. Providers vary in image format (URL, base64, tensor), max image count per turn, per-image token budget, and image preprocessing (resize, patchify, tiling). A single "image support" flag is not enough — the eval configuration must record what the adapter did to the image before it reached the model.
- **Task definition.** Adds an `image` field (or `images`, a list). For VQA, adds the multi-annotator reference-answer list. For grounding, adds bounding-box or segmentation ground truth.
- **Request type.** Generation, uniformly. Some early multimodal harnesses supported log-likelihood-over-answer-options for multiple-choice items; most current work uses generation and parses.
- **Scorer.** Family-dependent: VQA multi-annotator accuracy for short-answer, CLIPScore/BLIP-ITM for alignment, POPE-style precision/recall or a judge rubric for grounding.

The four-objects taxonomy still holds — the shape of the eval is the same as text-only — but the number of *knobs* is roughly doubled, and every one of them (image resolution passed to the model, preprocessing, max token budget, prompt template around the image) can move the reported number by 1–5 points on the same benchmark.

## Modality coverage: the argument specific to this family

The failure mode that separates multimodal eval from text-only eval is *coverage*. A text-only benchmark like MMLU has a fixed universe (57 subjects, mostly academic multiple choice). A user query in production might be outside that universe, but the benchmark's coverage of *its* universe is roughly complete. Multimodal benchmarks are different: the universe is "images," and every benchmark selects an aggressive slice of it.

Consider a partial taxonomy of vision:

- **Natural photographs** — VQAv2, OK-VQA, GQA. The vast majority of vision-benchmark data. COCO-derived.
- **Charts and plots** — ChartQA (Masry et al. 2022), PlotQA (Methani et al. 2020). Data-visualisation understanding.
- **Documents** — DocVQA (Mathew et al. 2021), InfographicVQA (Mathew et al. 2022). Document-layout understanding, OCR-heavy.
- **Text-in-scene** — TextVQA, ST-VQA. Reading text in natural images.
- **Diagrams** — AI2D (Kembhavi et al. 2016) for science-textbook diagrams; MMMU for domain-specific diagrams.
- **Math figures** — MathVista, MathVerse (Zhang et al. 2024).
- **UI screenshots** — ScreenQA, VisualWebBench (Liu et al. 2024).
- **Medical imaging** — RadVQA (varies), specialist datasets. Domain-restricted.
- **Satellite / remote sensing** — RSVQA (Lobry et al. 2020). Niche.
- **Multi-image reasoning** — NLVR2 (Suhr et al. 2019), MMMU multi-image subset.
- **Video** — a separate family entirely (see MSR-VTT, ActivityNet-QA, Video-MME). Out of scope for this chapter, but the same coverage argument applies.

The point of the enumeration is not that a serious eval suite must run all of them. The point is that "we measured our vision model on VQAv2 and MMMU" tells you the model works on natural photos and college-course multi-choice. It says nothing about charts, documents, screenshots, or diagrams. If your product is a spreadsheet copilot that reads screenshots of financial dashboards, VQAv2 is misleading — high on VQAv2 is compatible with total failure on your surface.

The eval-design implication:

- **Enumerate the visual surfaces your product actually encounters.** For each surface, either identify a benchmark that covers it or acknowledge the gap in the coverage section of your eval writeup.
- **Do not equate benchmark-column-coverage with capability coverage.** A leaderboard table showing scores on VQAv2, TextVQA, and MMMU does not span "vision"; it spans three named slices.
- **Design a bespoke slice for your surface.** 200–500 items sourced from real product inputs (with any PII scrubbed and licensing checked) is often more informative than any single public benchmark. mod-102 covers the sourcing discipline; this chapter is where that discipline shows up for images.

## Coverage-adjacent failure modes

Even within a single benchmark, coverage inside the benchmark matters:

- **English-only.** Almost every core VQA benchmark is English. Multilingual multimodal benchmarks (MaXM, xGQA) exist but are less commonly reported. A model that scores well on VQAv2 may fail on the same question asked in Portuguese.
- **Western-centric imagery.** COCO's images skew toward Western urban scenes. Objects and scenes common in other cultural contexts are underrepresented. If your product serves a global user base, cite the imagery-source geography.
- **Simple compositions.** Many benchmarks have short questions with one or two entities. Real user queries have longer instructions with multiple visual references, spatial constraints, and follow-ups. Multi-turn multimodal benchmarks (MMDU, MMBench-Video) probe this but are less mature.
- **Adult-oriented content and safety-critical categories.** These are typically filtered out of academic benchmarks. If your product must handle them (moderation, safety review), academic benchmarks *cannot* tell you if you're safe there.

## Practical multimodal-eval hygiene

- **Log image preprocessing.** Model providers apply different resize, crop, and tile strategies before the image reaches the model. Two adapters can produce different scores on the same benchmark; log the image resolution and any preprocessing the adapter applied.
- **Log per-image token budget.** Vision models charge per image tile; some benchmarks contain very large images that get down-scaled aggressively (losing text-in-scene detail). Confirm your adapter is not silently truncating.
- **Prompt sensitivity is worse for VLMs.** The chat template around an image (system prompt, image position in the message list, whether the image comes before or after the question) affects scores. Fix the template and log it, per mod-104 Chapter 7.
- **Judge-graded multimodal is even harder.** If your grader is a vision-language judge (a common pattern for open-ended visual tasks), you have compounded biases — image-token budget of the judge, judge's own vision blind spots, and self-preference across vision-language model families. Calibrate to human labels on a subset of items or the number is untethered.
- **Report per-slice accuracy.** MMMU has 30 subjects; VQAv2 has question-type slices. A single headline number hides which subject the model degrades on. mod-101's slice-first discipline applies unmodified.

## Summary

Multimodal eval has three canonical instruments — VQA-style short-answer accuracy (VQAv2, MMMU, GQA, TextVQA), image-text alignment (CLIPScore, BLIP ITM), and vision-grounded instruction following (POPE and rubric-graded judges) — each with a distinct measurement question and distinct failure modes. Beyond the instruments, the eval-design question specific to this family is *modality coverage*: benchmarks each select a narrow slice of the visual world, and coverage of the *product surface* has to be argued explicitly, not inherited from a leaderboard column. English-centric, Western-centric, natural-photo-centric benchmarks are the norm; documents, charts, screenshots, diagrams, medical, and remote-sensing are separate slices. A serious multimodal eval writes down the visual surfaces the product encounters, maps each to a benchmark or a bespoke slice, and reports per-slice accuracy plus preprocessing/prompt-template settings that would otherwise silently move the number. The final chapter is about the composition step that closes this argument: taking the four instruments in this module and the ones from mod-104 through mod-106, and building a suite for a specific product rather than a leaderboard collage.
