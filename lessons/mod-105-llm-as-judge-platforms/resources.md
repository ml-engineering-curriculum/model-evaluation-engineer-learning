# Resources for mod-105-llm-as-judge-platforms (LLM-as-Judge Platforms: Rubrics, Bias Controls, and Calibration to Humans)

These are the primary references behind the chapters. Prefer the original paper or the maintained tool repository over secondary write-ups.

## The LLM-as-judge setup itself

- Zheng et al. 2023. *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*. NeurIPS Datasets & Benchmarks. [arXiv:2306.05685](https://arxiv.org/abs/2306.05685). The paper that formalizes the modern LLM-as-judge setup — introduces MT-Bench (absolute mode) and the pairwise Chatbot Arena setup with GPT-4 as a proxy judge, and characterizes position, verbosity/length, and self-enhancement biases end-to-end. The baseline for everything else in this module.
- Chiang et al. 2024. *Chatbot Arena: An Open Platform for Evaluating LLMs by Human Preference*. ICML. [arXiv:2403.04132](https://arxiv.org/abs/2403.04132). The methodology paper behind the public Chatbot Arena leaderboard — crowdsourced pairwise voting, Bradley-Terry aggregation, sampling strategies, and the practical statistics of running a large open arena.
- LMSYS Chatbot Arena leaderboard and repositories: [lmarena.ai](https://lmarena.ai/), [`lm-sys/FastChat`](https://github.com/lm-sys/FastChat), and the [`lmarena/arena-hard-auto`](https://github.com/lmarena/arena-hard-auto) repo — the current reference implementation of a Bradley-Terry-based automatic-judge arena.

## Bias measurement and mitigation

- Panickssery, Bowman, and Feng. 2024. *LLM Evaluators Recognize and Favor Their Own Generations*. [arXiv:2404.13076](https://arxiv.org/abs/2404.13076). The primary reference for self-preference bias in LLM judges — measures self-recognition across GPT-4, Claude, and Llama-family models and shows the correlation with self-preference.
- Dubois et al. 2024. *Length-Controlled AlpacaEval: A Simple Way to Debias Automatic Evaluators*. [arXiv:2404.04475](https://arxiv.org/abs/2404.04475). The reference for length-controlled win rate (LC-WR) on pairwise judging, with a regression-based length adjustment applied to the AlpacaEval leaderboard.
- AlpacaEval repository and leaderboard: [`tatsu-lab/alpaca_eval`](https://github.com/tatsu-lab/alpaca_eval). Both raw and length-controlled win rates are published; the code is a working reference for the length-adjustment.
- Wang et al. 2024. *PandaLM: An Automatic Evaluation Benchmark for LLM Instruction Tuning Optimization*. ICLR. [arXiv:2306.05087](https://arxiv.org/abs/2306.05087). Early open-weights judge model that documents position-bias mitigations in its training data construction.

## Dedicated trained judges

- Kim et al. 2024. *Prometheus: Inducing Fine-Grained Evaluation Capability in Language Models*. ICLR. [arXiv:2310.08491](https://arxiv.org/abs/2310.08491). Prometheus 1 — a Llama-2-based dedicated absolute-scoring judge trained on rubric-shaped tasks.
- Kim et al. 2024. *Prometheus 2: An Open Source Language Model Specialized in Evaluating Other Language Models*. EMNLP. [arXiv:2405.01535](https://arxiv.org/abs/2405.01535). Prometheus 2, with pairwise-grading support and reported parity with GPT-4 on several rubric benchmarks at 7B and 8x7B scales.
- Prometheus repository and rubric-prompt library: [`prometheus-eval/prometheus-eval`](https://github.com/prometheus-eval/prometheus-eval). The native rubric format Prometheus expects — deviating from it costs κ.
- Zhu, Wang, and Wang. 2023. *JudgeLM: Fine-tuned Large Language Models are Scalable Judges*. [arXiv:2310.17631](https://arxiv.org/abs/2310.17631) — the JudgeLM paper and the [`baaivision/JudgeLM`](https://github.com/baaivision/JudgeLM) repo.
- Li et al. 2024. *Generative Judge for Evaluating Alignment* (Auto-J). ICLR. [arXiv:2310.05470](https://arxiv.org/abs/2310.05470) — dedicated critique-and-scoring judge trained on massive-scale scenario data.

## Agreement statistics and calibration

- Cohen, J. 1960. *A Coefficient of Agreement for Nominal Scales*. Educational and Psychological Measurement — Cohen's κ, primary reference.
- Landis, J.R. and Koch, G.G. 1977. *The Measurement of Observer Agreement for Categorical Data*. Biometrics 33(1) — the paper behind the "0.6 = substantial, 0.8 = near-perfect" κ interpretation bands (with the caveat that Landis and Koch themselves called the bands arbitrary anchors, not natural constants).
- Krippendorff, K. 2004. *Content Analysis: An Introduction to Its Methodology*. Sage — Krippendorff's α, the multi-rater / mixed-scale generalization of κ. See also the [`krippendorff` Python package](https://github.com/pln-fing-udelar/fast-krippendorff).
- scikit-learn `metrics` documentation: [`cohen_kappa_score`](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.cohen_kappa_score.html) (supports `weights="linear"` and `weights="quadratic"` for ordinal rubrics).
- scipy.stats documentation: [`spearmanr`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.spearmanr.html), [`kendalltau`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.kendalltau.html) — the continuous / ranking counterparts.
- Smith, P.C. and Kendall, L.M. 1963. *Retranslation of Expectations: An Approach to the Construction of Unambiguous Anchors for Rating Scales*. Journal of Applied Psychology — the industrial-organizational psychology origin of behaviorally-anchored rating scales (BARS), which is the underlying pattern for anchored rubrics.

## Rating models: Bradley-Terry and ELO

- Bradley, R.A. and Terry, M.E. 1952. *Rank Analysis of Incomplete Block Designs: I. The Method of Paired Comparisons*. Biometrika 39(3/4) — the original Bradley-Terry paper.
- Elo, A.E. 1978. *The Rating of Chessplayers, Past and Present*. Arco Publishing — the primary reference for the ELO rating system.
- Davidson, R.R. 1970. *On Extending the Bradley-Terry Model to Accommodate Ties in Paired Comparison Experiments*. Journal of the American Statistical Association 65(329) — the standard extension for tie-aware Bradley-Terry.
- The `choix` Python package: [`lucasmaystre/choix`](https://github.com/lucasmaystre/choix) — a maintained Bradley-Terry (and Plackett-Luce, and related pairwise-comparison models) inference library with cluster bootstrap.
- Boyd and Silk 1973 / Zermelo 1929 — the earlier statistical-inference machinery on paired comparisons that Bradley and Terry formalized.

## Judge platforms and tooling

- OpenAI evals: [`openai/evals`](https://github.com/openai/evals) — the [`modelgraded` spec docs](https://github.com/openai/evals/blob/main/docs/completion-fns.md) and the pairwise elsuites. Reference implementation of `cot_classify` and rubric-based ModelBasedClassify.
- EleutherAI [`lm-evaluation-harness`](https://github.com/EleutherAI/lm-evaluation-harness) — supports LLM-judge output types for tasks that produce free-form generations judged by a second model.
- UK AISI [Inspect](https://inspect.aisi.org.uk/) — `model_graded_qa` and custom `Scorer` primitives for judge-based scoring in a solver/scorer framework. Docs at [inspect.aisi.org.uk/scorers.html](https://inspect.aisi.org.uk/scorers.html).
- MT-Bench prompts and pipeline: [`lm-sys/FastChat` — `fastchat/llm_judge`](https://github.com/lm-sys/FastChat/tree/main/fastchat/llm_judge) — the reference rubric templates and judge-run scripts from the MT-Bench paper.
- HELM: [`stanford-crfm/helm`](https://github.com/stanford-crfm/helm) — includes model-graded scenarios and metrics.

## Adjacent literature worth being aware of

- Wang, Yao, et al. 2023. *Large Language Models are not Fair Evaluators*. [arXiv:2305.17926](https://arxiv.org/abs/2305.17926). Early systematic characterization of position bias in LLM judges and a proposal for calibration via order swapping.
- Wu and Aji. 2025. *Style Over Substance: Evaluation Biases for Large Language Models*. COLING. [arXiv:2307.03025](https://arxiv.org/abs/2307.03025). Documents that LLM judges over-weight surface style features (formality, formatting) relative to content correctness — the "style-bias" the module's exercises probe for.
- Liu et al. 2023. *G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment*. EMNLP. [arXiv:2303.16634](https://arxiv.org/abs/2303.16634). Alternative rubric-and-CoT framework for absolute NLG scoring; a good complement to MT-Bench-style rubrics.
- Verga et al. 2024. *Replacing Judges with Juries: Evaluating LLM Generations with a Panel of Diverse Models*. [arXiv:2404.18796](https://arxiv.org/abs/2404.18796). The empirical case for panel-of-judges: cheaper, more diverse panels of smaller judges often outperform a single frontier judge on agreement with humans.
