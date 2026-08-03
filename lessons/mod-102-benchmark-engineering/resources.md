# Resources for mod-102-benchmark-engineering

Primary sources for the material in this module. Every citation is either a paper, a textbook, a first-party standard, or an official documentation source. No secondary summaries.

## Data documentation, licensing, and provenance (Chapter 1)

- **Gebru, T., Morgenstern, J., Vecchione, B., Wortman Vaughan, J., Wallach, H., Daumé III, H., & Crawford, K. (2021).** "Datasheets for Datasets." *Communications of the ACM*, 64(12), 86–92. The standard for dataset documentation. [arXiv:1803.09010](https://arxiv.org/abs/1803.09010).
- **Bender, E. M., & Friedman, B. (2018).** "Data Statements for Natural Language Processing: Toward Mitigating System Bias and Enabling Better Science." *TACL*, 6, 587–604. Companion to Datasheets, NLP-specific. [Paper](https://aclanthology.org/Q18-1041/).
- **Pushkarna, M., Zaldivar, A., & Kjartansson, O. (2022).** "Data Cards: Purposeful and Transparent Dataset Documentation for Responsible AI." *FAccT.* Google's dataset-card standard. [arXiv:2204.01075](https://arxiv.org/abs/2204.01075).
- **Akhtar, M., Benjelloun, O., Conforti, C., et al. (2024).** "Croissant: A Metadata Format for ML-Ready Datasets." *MLCommons.* First-party spec + tooling for ML dataset metadata (license, provenance, structure). [Croissant docs](https://mlcommons.org/croissant/).
- **SPDX License List.** The canonical registry of SPDX license identifiers. [spdx.org/licenses](https://spdx.org/licenses/).
- **Creative Commons.** Human-readable and legal-code URLs for each CC license variant. [creativecommons.org/licenses](https://creativecommons.org/licenses/).
- **Stack Exchange licensing statement.** Documents the CC-BY-SA licensing of user contributions, including the version transition. [stackoverflow.com/help/licensing](https://stackoverflow.com/help/licensing).
- **Dodge, J., Sap, M., Marasović, A., Agnew, W., Ilharco, G., Groeneveld, D., Mitchell, M., & Gardner, M. (2021).** "Documenting Large Webtext Corpora: A Case Study on the Colossal Clean Crawled Corpus." *EMNLP.* A rigorous analysis of what is actually in a large webtext dataset. [arXiv:2104.08758](https://arxiv.org/abs/2104.08758).
- **Longpre, S., Mahari, R., Chen, A., Obeng-Marnu, N., et al. (2023).** "The Data Provenance Initiative: A Large Scale Audit of Dataset Licensing & Attribution in AI." *NeurIPS Datasets & Benchmarks.* Systematic audit of licensing and provenance across popular ML datasets. [arXiv:2310.16787](https://arxiv.org/abs/2310.16787).

## Dataset construction and schema (Chapter 2)

- **Hugging Face Datasets.** Reference implementation and format documentation for the schema you will most often ship benchmarks in. [huggingface.co/docs/datasets](https://huggingface.co/docs/datasets).
- **TensorFlow Datasets DatasetBuilder guide.** A different but equally instructive reference for reproducible dataset packaging. [tensorflow.org/datasets/add_dataset](https://www.tensorflow.org/datasets/add_dataset).
- **Apache Parquet format specification.** The Parquet file format is the recommended columnar storage for benchmark data. [github.com/apache/parquet-format](https://github.com/apache/parquet-format).
- **Kirchenbauer, J., Geiping, J., Wen, Y., Katz, J., Miers, I., & Goldstein, T. (2023).** "A Watermark for Large Language Models." *ICML.* Not benchmark-specific but useful for the schema-level question of how to embed detectability into text artifacts. [arXiv:2301.10226](https://arxiv.org/abs/2301.10226).

## Annotation instructions and pilots (Chapter 3)

- **Snow, R., O'Connor, B., Jurafsky, D., & Ng, A. Y. (2008).** "Cheap and Fast — but is it Good? Evaluating Non-Expert Annotations for Natural Language Tasks." *EMNLP.* Foundational study on crowd-annotator agreement and cost trade-offs. [Paper](https://aclanthology.org/D08-1027/).
- **Bragg, J., Mausam, & Weld, D. S. (2018).** "Sprout: Crowd-Powered Task Design for Crowdsourcing." *UIST.* Systematizes iterative instruction refinement with pilots. [arXiv:1707.08154](https://arxiv.org/abs/1707.08154).
- **Nangia, N., Sugawara, S., Trivedi, H., Warstadt, A., Vania, C., & Bowman, S. R. (2021).** "What Ingredients Make for an Effective Crowdsourcing Protocol for Difficult NLU Data Collection Tasks?" *ACL.* Empirical study of protocol choices for hard NLU annotation. [arXiv:2106.00794](https://arxiv.org/abs/2106.00794).
- **Northcutt, C. G., Athalye, A., & Mueller, J. (2021).** "Pervasive Label Errors in Test Sets Destabilize Machine Learning Benchmarks." *NeurIPS Datasets & Benchmarks.* Argues that label errors in canonical test sets change model rankings; motivates the pilot + adjudication discipline. [arXiv:2103.14749](https://arxiv.org/abs/2103.14749).

## Inter-annotator agreement and adjudication (Chapter 4)

- **Cohen, J. (1960).** "A Coefficient of Agreement for Nominal Scales." *Educational and Psychological Measurement*, 20(1), 37–46. The original Cohen's κ paper.
- **Cohen, J. (1968).** "Weighted kappa: Nominal scale agreement with provision for scaled disagreement or partial credit." *Psychological Bulletin*, 70(4), 213–220. The weighted-κ paper.
- **Fleiss, J. L. (1971).** "Measuring nominal scale agreement among many raters." *Psychological Bulletin*, 76(5), 378–382. The Fleiss κ paper.
- **Krippendorff, K. (2004).** *Content Analysis: An Introduction to Its Methodology* (2nd ed.), Chapter 11. Sage. The canonical treatment of Krippendorff's α, including missing-data and multi-annotator cases.
- **Landis, J. R., & Koch, G. G. (1977).** "The Measurement of Observer Agreement for Categorical Data." *Biometrics*, 33(1), 159–174. The (often over-cited) benchmarks table for interpreting κ magnitudes.
- **Feinstein, A. R., & Cicchetti, D. V. (1990).** "High agreement but low kappa: I. The problems of two paradoxes." *Journal of Clinical Epidemiology*, 43(6), 543–549. The kappa-paradox paper; explains why raw agreement and κ can diverge under imbalance.
- **Byrt, T., Bishop, J., & Carlin, J. B. (1993).** "Bias, prevalence and kappa." *Journal of Clinical Epidemiology*, 46(5), 423–429. Follow-up analysis on kappa's prevalence sensitivity.
- **Artstein, R., & Poesio, M. (2008).** "Inter-Coder Agreement for Computational Linguistics." *Computational Linguistics*, 34(4), 555–596. The go-to survey of IAA statistics in NLP with practical recommendations. [Paper](https://direct.mit.edu/coli/article/34/4/555/1988).
- **`sklearn.metrics.cohen_kappa_score`** — Cohen's κ with linear/quadratic weighting options. [Docs](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.cohen_kappa_score.html).
- **`statsmodels.stats.inter_rater.fleiss_kappa`** — Fleiss' κ. [Docs](https://www.statsmodels.org/stable/generated/statsmodels.stats.inter_rater.fleiss_kappa.html).
- **`krippendorff`** Python package — Krippendorff's α with support for missing values and multiple measurement levels. [PyPI](https://pypi.org/project/krippendorff/).

## Benchmark contamination detection (Chapter 5)

- **Brown, T. B., Mann, B., Ryder, N., et al. (2020).** "Language Models are Few-Shot Learners." *NeurIPS* (GPT-3 paper). Section 4 describes the 13-gram overlap contamination filter used to construct a clean-subset evaluation. [arXiv:2005.14165](https://arxiv.org/abs/2005.14165).
- **Chowdhery, A., Narang, S., Devlin, J., et al. (2022).** "PaLM: Scaling Language Modeling with Pathways." Discusses contamination filtering and per-benchmark contamination analyses. [arXiv:2204.02311](https://arxiv.org/abs/2204.02311).
- **Sainz, O., Campos, J. A., García-Ferrero, I., Etxaniz, J., López de Lacalle, O., & Agirre, E. (2023).** "NLP Evaluation in Trouble: On the Need to Measure LLM Data Contamination for each Benchmark." *EMNLP Findings.* Contamination taxonomy and per-benchmark measurement argument. [arXiv:2310.18018](https://arxiv.org/abs/2310.18018).
- **Golchin, S., & Surdeanu, M. (2023).** "Time Travel in LLMs: Tracing Data Contamination in Large Language Models." Quiz-completion-style probes for detecting contamination without training-corpus access. [arXiv:2308.08493](https://arxiv.org/abs/2308.08493).
- **Oren, Y., Meister, N., Chatterji, N., Ladhak, F., & Hashimoto, T. B. (2023).** "Proving Test Set Contamination in Black Box Language Models." Order-based statistical probe with a provable false-positive guarantee. [arXiv:2310.17623](https://arxiv.org/abs/2310.17623).
- **Shi, W., Ajith, A., Xia, M., Huang, Y., Liu, D., Song, T., Zhang, J., Zettlemoyer, L., & Chen, D. (2024).** "Detecting Pretraining Data from Large Language Models." Introduces the Min-K% Prob membership-inference attack. [arXiv:2310.16789](https://arxiv.org/abs/2310.16789).
- **Deng, C., Zhao, Y., Tang, X., Gerstein, M., & Cohan, A. (2024).** "Investigating Data Contamination in Modern Benchmarks for Large Language Models." Empirical study across widely-used benchmarks. [arXiv:2311.09783](https://arxiv.org/abs/2311.09783).
- **Xu, R., Feng, Z., Xu, Y., Cheng, Z., Jia, R., & Yang, D. (2024).** "Benchmark Data Contamination of Large Language Models: A Survey." Recent survey summarizing the detector families. [arXiv:2406.04244](https://arxiv.org/abs/2406.04244).
- **Elazar, Y., Bhagia, A., Magnusson, I., et al. (2024).** "What's In My Big Data?" *ICLR.* Tooling and analysis for auditing pretraining corpora — the corpus-side counterpart to per-benchmark contamination probes. [arXiv:2310.20707](https://arxiv.org/abs/2310.20707).
- **infini-gram.** Public n-gram indices over major pretraining corpora (RedPajama, Pile, C4). [infini-gram.io](https://infini-gram.io/).
- **`datasketch`** Python library — Bloom filters, MinHash, and LSH implementations useful for surface-overlap detection at scale. [ekzhu.github.io/datasketch](https://ekzhu.github.io/datasketch/).

## Benchmark versioning and deprecation (Chapter 6)

- **Semantic Versioning 2.0.0.** [semver.org](https://semver.org/). The specification the chapter adapts to benchmarks.
- **Mitchell, M., Wu, S., Zaldivar, A., Barnes, P., Vasserman, L., Hutchinson, B., Spitzer, E., Raji, I. D., & Gebru, T. (2019).** "Model Cards for Model Reporting." *FAT\*.* The reporting-side complement to the versioning discipline. [arXiv:1810.03993](https://arxiv.org/abs/1810.03993).
- **Bowman, S. R., & Dahl, G. E. (2021).** "What Will It Take to Fix Benchmarking in Natural Language Understanding?" *NAACL.* Argues for stronger benchmark maintenance and rotation practices. [arXiv:2104.02145](https://arxiv.org/abs/2104.02145).
- **Recht, B., Roelofs, R., Schmidt, L., & Shankar, V. (2019).** "Do ImageNet Classifiers Generalize to ImageNet?" *ICML.* Canonical demonstration of the score-inflation that motivates a fresh-holdout rotation practice. [arXiv:1902.10811](https://arxiv.org/abs/1902.10811).
- **Koch, B., Denton, E., Hanna, A., & Foster, J. G. (2021).** "Reduced, Reused and Recycled: The Life of a Dataset in Machine Learning Research." *NeurIPS Datasets & Benchmarks.* Empirical study of benchmark reuse patterns and its costs. [arXiv:2112.01716](https://arxiv.org/abs/2112.01716).
- **Sigstore.** Cryptographic signing tooling for artifacts (including datasets and model artifacts). [sigstore.dev](https://www.sigstore.dev/).

## Public / private holdouts and canaries (Chapter 7)

- **Srivastava, A., Rastogi, A., Rao, A., et al. (2022).** "Beyond the Imitation Game: Quantifying and Extrapolating the Capabilities of Language Models" (BIG-bench). *arXiv/TMLR.* Documents the BIG-bench canary string, one of the earliest widely-cited benchmark canaries. [arXiv:2206.04615](https://arxiv.org/abs/2206.04615).
- **Kaggle "Rules and Best Practices" for held-out evaluation.** [kaggle.com/docs/competitions](https://www.kaggle.com/docs/competitions). Operational precedent for private test sets in a large public leaderboard.
- **DynaBench (Kiela, D., Bartolo, M., Nie, Y., et al. 2021).** "Dynabench: Rethinking Benchmarking in NLP." *NAACL.* Argues for dynamic, rotated benchmarks that keep the private test set moving. [arXiv:2104.14337](https://arxiv.org/abs/2104.14337).
- **Carlini, N., Tramèr, F., Wallace, E., Jagielski, M., Herbert-Voss, A., Lee, K., Roberts, A., Brown, T., Song, D., Erlingsson, U., Oprea, A., & Raffel, C. (2021).** "Extracting Training Data from Large Language Models." *USENIX Security.* Foundational work on model-behavioral extraction of memorized content — the mechanism canary probes exploit. [arXiv:2012.07805](https://arxiv.org/abs/2012.07805).
- **Carlini, N., Ippolito, D., Jagielski, M., Lee, K., Tramèr, F., & Zhang, C. (2023).** "Quantifying Memorization Across Neural Language Models." *ICLR.* Empirical characterization of when models memorize training data — informs canary token entropy and placement. [arXiv:2202.07646](https://arxiv.org/abs/2202.07646).
- **Ippolito, D., Yu, W., Duckworth, D., et al. (2023).** "Preventing Verbatim Memorization in Language Models Gives a False Sense of Privacy." *INLG.* Discusses the limits of memorization-based leakage detection; important context for canary probe design. [arXiv:2210.17546](https://arxiv.org/abs/2210.17546).
- **Nasr, M., Carlini, N., Hayase, J., Jagielski, M., Cooper, A. F., Ippolito, D., Choquette-Choo, C. A., Wallace, E., Tramèr, F., & Lee, K. (2023).** "Scalable Extraction of Training Data from (Production) Language Models." Demonstrates practical extraction at scale — relevant for private-canary probe design. [arXiv:2311.17035](https://arxiv.org/abs/2311.17035).

## Standards and frameworks referenced across the module

- **NIST AI Risk Management Framework (AI RMF 1.0), 2023.** [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework). The "Measure" function is the direct interface for benchmark-engineering practice.
- **ISO/IEC 25059:2023,** Systems and software Quality Requirements and Evaluation (SQuaRE) — Quality model for AI systems. [ISO catalogue entry](https://www.iso.org/standard/80655.html).
- **ISO/IEC 5259 series (Data quality for analytics and ML).** Multi-part standard covering data-quality frameworks, measures, processes, and management. [ISO catalogue](https://www.iso.org/standard/81088.html).
- **EU AI Act (Regulation (EU) 2024/1689),** especially the obligations for high-risk systems and general-purpose AI models with systemic risk. [EUR-Lex text](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).
- **MLCommons benchmark governance model.** [mlcommons.org](https://mlcommons.org/). A working example of multi-stakeholder benchmark rotation and deprecation policy in production.
