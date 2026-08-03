# Resources for mod-106-human-evaluation (Human Evaluation: Annotator Workflows, Agreement, and Gold Sets)

Primary references and tooling for the module, grouped by chapter. Prefer these over blog-post summaries; the primary sources are where the definitions, caveats, and edge cases live.

## Foundational papers on inter-annotator agreement (Chapter 4)

- **Cohen, J. (1960). "A coefficient of agreement for nominal scales."** *Educational and Psychological Measurement* 20(1), 37–46. The original Cohen's κ paper.
- **Cohen, J. (1968). "Weighted kappa: Nominal scale agreement provision for scaled disagreement or partial credit."** *Psychological Bulletin* 70(4), 213–220. The weighted-κ extension for ordinal scales.
- **Fleiss, J.L. (1971). "Measuring nominal scale agreement among many raters."** *Psychological Bulletin* 76(5), 378–382. The multi-rater κ extension.
- **Krippendorff, K. (2018).** *Content Analysis: An Introduction to Its Methodology* (4th ed.). SAGE Publications. The book chapters on α are the canonical reference; earlier editions contain the same core material. See also Krippendorff's methodology reports at <https://www.asc.upenn.edu/sites/default/files/2021-03/Computing%20Krippendorff%27s%20Alpha-Reliability.pdf> for a compact worked introduction.
- **Landis, J.R., and Koch, G.G. (1977). "The measurement of observer agreement for categorical data."** *Biometrics* 33(1), 159–174. The source of the widely-cited (and widely-caveated) κ interpretation bands.
- **Feinstein, A.R., and Cicchetti, D.V. (1990). "High agreement but low kappa: I. The problems of two paradoxes."** *Journal of Clinical Epidemiology* 43(6), 543–549. The "kappa paradox" paper on prevalence sensitivity — required reading for anyone reporting κ on skewed distributions.
- **Byrt, T., Bishop, J., and Carlin, J.B. (1993). "Bias, prevalence and kappa."** *Journal of Clinical Epidemiology* 46(5), 423–429. Defines PABAK (Prevalence-Adjusted Bias-Adjusted Kappa) as an alternative to raw κ.
- **Artstein, R., and Poesio, M. (2008). "Inter-coder agreement for computational linguistics."** *Computational Linguistics* 34(4), 555–596. The canonical NLP-oriented survey of inter-annotator agreement statistics; recommended over any single primary paper as an orientation.

## Software libraries for agreement statistics (Chapter 4)

- **scikit-learn** — `sklearn.metrics.cohen_kappa_score` (Cohen's κ, weighted κ via the `weights` argument). <https://scikit-learn.org/stable/modules/generated/sklearn.metrics.cohen_kappa_score.html>
- **statsmodels** — `statsmodels.stats.inter_rater.fleiss_kappa` and `aggregate_raters`. <https://www.statsmodels.org/stable/generated/statsmodels.stats.inter_rater.fleiss_kappa.html>
- **krippendorff (PyPI)** — a small, well-maintained Krippendorff's α implementation supporting nominal / ordinal / interval / ratio levels of measurement and missing data. <https://pypi.org/project/krippendorff/>
- **scipy.stats** — `spearmanr` and `kendalltau` for continuous-agreement cases. <https://docs.scipy.org/doc/scipy/reference/stats.html>

## Human evaluation methodology in NLP and LLM eval (Chapters 1, 3, 6)

- **Zheng, L., Chiang, W.-L., Sheng, Y., Zhuang, S., Wu, Z., Zhuang, Y., ... Stoica, I. (2023). "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena."** NeurIPS 2023. <https://arxiv.org/abs/2306.05685> Formalizes the modern LLM-as-judge setup and includes the pairwise human study whose UX and analysis this module's Chapter 6 draws on.
- **Chiang, W.-L., Zheng, L., Sheng, Y., Angelopoulos, A.N., Li, T., Li, D., ... Stoica, I. (2024). "Chatbot Arena: An Open Platform for Evaluating LLMs by Human Preference."** ICML 2024. <https://arxiv.org/abs/2403.04132> The Chatbot Arena methodology paper — pairwise UX, Bradley-Terry aggregation, sampling strategies.
- **Veselovsky, V., Ribeiro, M.H., and West, R. (2023). "Artificial Artificial Artificial Intelligence: Crowd Workers Widely Use Large Language Models for Text Production Tasks."** <https://arxiv.org/abs/2306.07899> The estimate that 33–46% of MTurk workers on text-summarization tasks used LLMs — the primary source Chapter 6 cites for the bot-detection framing.
- **Clark, E., August, T., Serrano, S., Haduong, N., Gururangan, S., and Smith, N.A. (2021). "All That's 'Human' Is Not Gold: Evaluating Human Evaluation of Generated Text."** ACL 2021. <https://aclanthology.org/2021.acl-long.565/> A pointed study of how the design of a human evaluation determines what the resulting "human judgment" actually measures. Recommended for anyone about to run their first side-by-side study.
- **Karpinska, M., Akoury, N., and Iyyer, M. (2021). "The Perils of Using Mechanical Turk to Evaluate Open-Ended Text Generation."** EMNLP 2021. <https://aclanthology.org/2021.emnlp-main.97/> A specific case study on how a naïve crowd setup produces unreliable labels on open-ended generation tasks.

## Annotation tooling (Chapter 2, 6)

- **Label Studio** — an open-source, self-hostable data-labelling platform supporting text, image, audio, video, and time-series tasks. <https://labelstud.io/> Configurable via XML-defined interfaces; supports gold items, review queues, per-annotator agreement.
- **Prodigy** — commercial, scriptable annotation tool from Explosion, closely integrated with spaCy. <https://prodi.gy/> Strong for iterative active-learning workflows.
- **Argilla** — open-source annotation platform aimed at LLM-era workflows (feedback datasets, comparisons, custom fields). <https://argilla.io/>
- **Doccano** — lightweight open-source text-annotation tool; simplest to self-host for small teams. <https://github.com/doccano/doccano>
- **INCEpTION** — research-grade annotation platform out of the Technische Universität Darmstadt; strong on span, coreference, and linguistic annotation tasks. <https://inception-project.github.io/>

## Crowd platforms and vendors (Chapter 7)

Each of these vendors publishes their own documentation on task design, quality control, and worker vetting. Prefer their docs over third-party writeups when planning a real project.

- **Prolific** — academic-friendly paid crowd platform. <https://www.prolific.com/> See their researcher help-centre for task-design and quality-assurance guidance.
- **Scale AI** (including Remotasks / Outlier) — managed data-labelling for AI, including RLHF preference labelling. <https://scale.com/>
- **Surge AI** — managed data-labelling with a stated focus on data quality for LLM applications. <https://www.surgehq.ai/>
- **Mercor** — talent marketplace matching AI companies with domain-expert workers. <https://mercor.com/>
- **Appen** — long-established data-labelling vendor with global crowd operations. <https://appen.com/>
- **Toloka** — crowd-labelling platform with a research-friendly SDK and public benchmarks. <https://toloka.ai/>

## Data residency and compliance

- **GDPR (EU General Data Protection Regulation)** — official text and guidance at <https://gdpr-info.eu/> and the European Commission's data-protection portal <https://commission.europa.eu/law/law-topic/data-protection_en>. The specific relevance to annotation is Chapter V (transfers of personal data to third countries).
- **HIPAA** — the US Department of Health and Human Services HIPAA overview at <https://www.hhs.gov/hipaa/index.html>. Business Associate Agreements (BAAs) are the mechanism through which a data-labelling vendor becomes HIPAA-processable.
- **ITAR / EAR** — US export-control regimes governing technical data. The Directorate of Defense Trade Controls (ITAR): <https://www.pmddtc.state.gov/>. The Bureau of Industry and Security (EAR): <https://www.bis.doc.gov/>.

## Datasets useful for exercises

- **`lmsys/lmsys-arena-human-preference-55k`** — 55k pairwise human preference votes from Chatbot Arena, released as a Hugging Face dataset. <https://huggingface.co/datasets/lmsys/lmsys-arena-human-preference-55k>
- **`ucberkeley-dlab/measuring-hate-speech`** — a multi-annotator hate-speech dataset with per-rater labels, useful for demonstrating multi-rater agreement. <https://huggingface.co/datasets/ucberkeley-dlab/measuring-hate-speech>
- **Alpaca Eval / AlpacaFarm** — human and simulated preference data for instruction-following. <https://github.com/tatsu-lab/alpaca_eval>
- **MT-Bench** — the multi-turn dialog benchmark from the Zheng et al. 2023 paper, with the accompanying human judgments. <https://huggingface.co/datasets/lmsys/mt_bench_human_judgments>

## Broader reading

- **Bhardwaj, A., et al. "The Data Provenance Initiative."** <https://www.dataprovenance.org/> A community effort to audit dataset licences and provenance — cross-references the mod-102 provenance material and applies to human-labelled data as well.
- **Vaughan, J.W. (2018). "Making Better Use of the Crowd: How Crowdsourcing Can Advance Machine Learning Research."** *JMLR* 18(193), 1–46. <https://jmlr.org/papers/v18/17-234.html> A survey of crowd-annotation-for-ML methodology; older but foundational.
- **Northcutt, C., Athalye, A., and Mueller, J. (2021). "Pervasive Label Errors in Test Sets Destabilize Machine Learning Benchmarks."** NeurIPS Datasets and Benchmarks. <https://arxiv.org/abs/2103.14749> The reason gold-set and adjudication discipline (Chapter 5) matters — sloppy labels on a benchmark corrupt every downstream comparison.

Vendor cost figures, worker pay rates, and platform capability claims in the module chapters are directionally accurate at time of writing but change frequently; always verify against current vendor documentation before quoting to a stakeholder.
