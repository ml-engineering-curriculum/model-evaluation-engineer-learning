# Resources for mod-101-evaluation-foundations

Primary sources for the material in this module. Every citation is either a paper, a textbook, a first-party library page, or an official documentation source. No secondary summaries.

## Validity (Chapter 2)

- **Cronbach, L. J., & Meehl, P. E. (1955).** "Construct validity in psychological tests." *Psychological Bulletin*, 52(4), 281–302. The foundational construct-validity paper; the vocabulary is unchanged 70 years later.
- **Campbell, D. T., & Stanley, J. C. (1963).** *Experimental and Quasi-Experimental Designs for Research.* Rand McNally. Origin of the internal/external validity split.
- **Cook, T. D., & Campbell, D. T. (1979).** *Quasi-Experimentation: Design and Analysis Issues for Field Settings.* Houghton Mifflin. Adds statistical conclusion validity as a fourth frame; still the reference text.
- **Recht, B., Roelofs, R., Schmidt, L., & Shankar, V. (2019).** "Do ImageNet Classifiers Generalize to ImageNet?" *ICML.* Canonical demonstration of external-validity failure in machine learning benchmarks. [arXiv:1902.10811](https://arxiv.org/abs/1902.10811).
- **Torralba, A., & Efros, A. A. (2011).** "Unbiased Look at Dataset Bias." *CVPR.* Classic on dataset-specific idiosyncrasies that inflate cross-dataset generalization estimates.
- **Bowman, S. R., & Dahl, G. E. (2021).** "What Will It Take to Fix Benchmarking in Natural Language Understanding?" *NAACL.* Modern statement of the validity problems in NLP benchmarks. [arXiv:2104.02145](https://arxiv.org/abs/2104.02145).
- **Ribeiro, M. T., Wu, T., Guestrin, C., & Singh, S. (2020).** "Beyond Accuracy: Behavioral Testing of NLP Models with CheckList." *ACL.* Argues for capability-specific test suites over aggregate accuracy. Best-paper award. [arXiv:2005.04118](https://arxiv.org/abs/2005.04118).
- **Raji, I. D., Bender, E. M., Paullada, A., Denton, E., & Hanna, A. (2021).** "AI and the Everything in the Whole Wide World Benchmark." *NeurIPS Datasets and Benchmarks Track.* Critique of construct validity in general-purpose benchmarks. [arXiv:2111.15366](https://arxiv.org/abs/2111.15366).
<!-- needs-research: locate a well-cited cross-ML meta-review of eval failures to sit alongside Raji et al. 2021; candidates to check on the next research cycle include Blodgett et al. 2021 "Stereotyping Norwegian Salmon" (fairness-benchmark critique) and Koch et al. 2021 "Reduced, Reused and Recycled" (dataset reuse in NeurIPS D&B). -->

## Binomial CIs (Chapter 3)

- **Wilson, E. B. (1927).** "Probable Inference, the Law of Succession, and Statistical Inference." *Journal of the American Statistical Association*, 22(158), 209–212. The original score-interval paper.
- **Agresti, A., & Coull, B. A. (1998).** "Approximate is Better than 'Exact' for Interval Estimation of Binomial Proportions." *The American Statistician*, 52(2), 119–126. Systematic coverage comparison across intervals; recommends Wilson and the "adjusted-Wald" as defaults.
- **Brown, L. D., Cai, T. T., & DasGupta, A. (2001).** "Interval Estimation for a Binomial Proportion." *Statistical Science*, 16(2), 101–133. Companion paper to Agresti–Coull with a broader recommendation.
- **Clopper, C. J., & Pearson, E. S. (1934).** "The Use of Confidence or Fiducial Limits Illustrated in the Case of the Binomial." *Biometrika*, 26(4), 404–413. The exact ("Clopper–Pearson") interval.
- **`statsmodels.stats.proportion.proportion_confint`** — reference implementation for Wilson, Agresti–Coull, Clopper–Pearson, Jeffreys, and Wald. [Docs](https://www.statsmodels.org/stable/generated/statsmodels.stats.proportion.proportion_confint.html).
- **`scipy.stats.binomtest(...).proportion_ci`** — reference implementation for Wilson, Clopper–Pearson (exact), and other methods. [Docs](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.binomtest.html).

## Bootstrap (Chapter 4)

- **Efron, B. (1979).** "Bootstrap Methods: Another Look at the Jackknife." *The Annals of Statistics*, 7(1), 1–26. The original bootstrap paper.
- **Efron, B., & Tibshirani, R. J. (1993).** *An Introduction to the Bootstrap.* Chapman & Hall. The standard book-length treatment; Ch. 14 covers BCa, Ch. 16 covers bootstrap hypothesis testing.
- **Davison, A. C., & Hinkley, D. V. (1997).** *Bootstrap Methods and Their Application.* Cambridge University Press. More technical companion to Efron & Tibshirani; strong on clustering and dependence.
- **Efron, B. (1987).** "Better Bootstrap Confidence Intervals." *Journal of the American Statistical Association*, 82(397), 171–185. The BCa paper.
- **`scipy.stats.bootstrap`** — vectorized reference implementation with percentile, basic, and BCa methods. [Docs](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.bootstrap.html).

## Paired comparison tests (Chapter 5)

- **McNemar, Q. (1947).** "Note on the Sampling Error of the Difference Between Correlated Proportions or Percentages." *Psychometrika*, 12(2), 153–157. The original McNemar test.
- **Dietterich, T. G. (1998).** "Approximate Statistical Tests for Comparing Supervised Classification Learning Algorithms." *Neural Computation*, 10(7), 1895–1923. The canonical ML paper on paired classifier comparison; recommends McNemar and the 5×2 CV `t`-test.
- **Edgington, E. S. (1995).** *Randomization Tests* (3rd ed.). Marcel Dekker. Comprehensive on permutation and sign tests.
- **Fleiss, J. L., Levin, B., & Paik, M. C. (2003).** *Statistical Methods for Rates and Proportions* (3rd ed.). Wiley. Standard reference on the exact and mid-p McNemar tests.
- **`statsmodels.stats.contingency_tables.mcnemar`** — reference implementation for McNemar with `exact` and `correction` options. [Docs](https://www.statsmodels.org/stable/generated/statsmodels.stats.contingency_tables.mcnemar.html).
- **Cohen, J. (1988).** *Statistical Power Analysis for the Behavioral Sciences* (2nd ed.). Lawrence Erlbaum. The reference on power calculations; the sample-size formulas are unchanged.

## Multiple comparisons and FDR (Chapter 6)

- **Benjamini, Y., & Hochberg, Y. (1995).** "Controlling the False Discovery Rate: A Practical and Powerful Approach to Multiple Testing." *Journal of the Royal Statistical Society: Series B*, 57(1), 289–300. The BH procedure.
- **Benjamini, Y., & Yekutieli, D. (2001).** "The Control of the False Discovery Rate in Multiple Testing under Dependency." *The Annals of Statistics*, 29(4), 1165–1188. The BY variant that controls FDR under arbitrary dependence.
- **Storey, J. D. (2002).** "A Direct Approach to False Discovery Rates." *JRSS-B*, 64(3), 479–498. The `q`-value framing.
- **Gelman, A., & Loken, E. (2013).** "The Garden of Forking Paths." Working paper, later published in a shortened form as "The Statistical Crisis in Science," *American Scientist*, 2014. Foundational on undisclosed-flexibility multiplicity. [Original working paper](http://www.stat.columbia.edu/~gelman/research/unpublished/p_hacking.pdf).
- **`statsmodels.stats.multitest.multipletests`** — reference implementation for Bonferroni, Holm, BH, BY, and other corrections. [Docs](https://www.statsmodels.org/stable/generated/statsmodels.stats.multitest.multipletests.html).
- **`scipy.stats.false_discovery_control`** — SciPy's BH / BY implementation. [Docs](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.false_discovery_control.html).

## Reading eval reports (Chapter 7)

- **Mitchell, M., Wu, S., Zaldivar, A., Barnes, P., Vasserman, L., Hutchinson, B., Spitzer, E., Raji, I. D., & Gebru, T. (2019).** "Model Cards for Model Reporting." *FAT\*.* The model-card standard. [arXiv:1810.03993](https://arxiv.org/abs/1810.03993).
- **Gebru, T., Morgenstern, J., Vecchione, B., Wortman Vaughan, J., Wallach, H., Daumé III, H., & Crawford, K. (2018).** "Datasheets for Datasets." *arXiv.* Companion standard for the eval-set side. [arXiv:1803.09010](https://arxiv.org/abs/1803.09010).
- **Liang, P., Bommasani, R., et al. (2022).** "Holistic Evaluation of Language Models" (HELM). *arXiv.* Multi-dimensional eval methodology; the paper is worth reading in full as a model of what a reproducible eval report looks like. [arXiv:2211.09110](https://arxiv.org/abs/2211.09110).
- **Zheng, L., Chiang, W.-L., et al. (2023).** "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena." *NeurIPS.* Foundational reference on judge biases and Arena-style ELO. [arXiv:2306.05685](https://arxiv.org/abs/2306.05685).
- **Contamination detection primers.** For the state of the art on detecting benchmark contamination in LLMs, see:
    - **Golchin, S., & Surdeanu, M. (2023).** "Time Travel in LLMs: Tracing Data Contamination in Large Language Models." [arXiv:2308.08493](https://arxiv.org/abs/2308.08493).
    - **Sainz, O., Campos, J. A., García-Ferrero, I., Etxaniz, J., López de Lacalle, O., & Agirre, E. (2023).** "NLP Evaluation in Trouble: On the Need to Measure LLM Data Contamination for each Benchmark." *EMNLP Findings.* [arXiv:2310.18018](https://arxiv.org/abs/2310.18018).
    - <!-- needs-research: verify the arXiv ID and exact author list for Deng et al. 2023 "Investigating Data Contamination in Modern Benchmarks for Large Language Models" before citing. -->

## Standards and frameworks referenced across the module

- **NIST AI Risk Management Framework (AI RMF 1.0), 2023.** [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework). The "Measure" function is the direct interface with evaluation engineering.
- **ISO/IEC 25059:2023,** Systems and software Quality Requirements and Evaluation (SQuaRE) — Quality model for AI systems. [ISO catalogue entry](https://www.iso.org/standard/80655.html).
- **EU AI Act (Regulation (EU) 2024/1689),** particularly the obligations for high-risk systems and general-purpose AI models with systemic risk. [EUR-Lex text](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).

Later modules (particularly mod-102, mod-109, mod-110, mod-112) go deep on these; the entries above are the primary sources you should have on hand for this module's exercises.
