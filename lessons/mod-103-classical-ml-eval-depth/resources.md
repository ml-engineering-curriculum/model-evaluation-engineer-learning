# Resources for mod-103-classical-ml-eval-depth (Classical ML Evaluation Depth: Calibration, Slicing, and Fairness)

Primary sources for the material in this module. Every citation is either a paper, a textbook, a first-party library page, or an official documentation / legal source. No secondary summaries.

## Slicing and per-slice reporting (Chapter 1)

- **Mitchell, M., Wu, S., Zaldivar, A., Barnes, P., Vasserman, L., Hutchinson, B., Spitzer, E., Raji, I. D., & Gebru, T. (2019).** "Model Cards for Model Reporting." *FAT\*.* The model-card standard, which lists disaggregated per-group evaluation as a required section. [arXiv:1810.03993](https://arxiv.org/abs/1810.03993).
- **Simpson, E. H. (1951).** "The Interpretation of Interaction in Contingency Tables." *Journal of the Royal Statistical Society: Series B*, 13(2), 238–241. The original description of the aggregation-reversal effect that carries his name.
- **Blyth, C. R. (1972).** "On Simpson's Paradox and the Sure-Thing Principle." *Journal of the American Statistical Association*, 67(338), 364–366. The rigorous statement of the paradox and its decision-theoretic implications.
- **Chen, I., Johansson, F. D., & Sontag, D. (2018).** "Why Is My Classifier Discriminatory?" *NeurIPS.* Formal decomposition of aggregate performance gaps into base-rate, model-capacity, and sample-size contributions per subgroup. [arXiv:1805.12002](https://arxiv.org/abs/1805.12002).
- **Sagawa, S., Koh, P. W., Hashimoto, T. B., & Liang, P. (2020).** "Distributionally Robust Neural Networks for Group Shifts." *ICLR.* Group-DRO training; the evaluation methodology (worst-group vs. average) is directly reusable in per-slice reporting. [arXiv:1911.08731](https://arxiv.org/abs/1911.08731).

## Calibration measurement (Chapter 2)

- **Guo, C., Pleiss, G., Sun, Y., & Weinberger, K. Q. (2017).** "On Calibration of Modern Neural Networks." *ICML.* The paper that established that modern deep classifiers are systematically overconfident and popularized temperature scaling. [arXiv:1706.04599](https://arxiv.org/abs/1706.04599).
- **Naeini, M. P., Cooper, G. F., & Hauskrecht, M. (2015).** "Obtaining Well Calibrated Probabilities Using Bayesian Binning." *AAAI.* Introduces the binning-based Expected Calibration Error formulation now standard in the ML literature. [Paper (AAAI)](https://ojs.aaai.org/index.php/AAAI/article/view/9602).
- **Murphy, A. H. (1973).** "A New Vector Partition of the Probability Score." *Journal of Applied Meteorology*, 12(4), 595–600. The Brier-score decomposition into reliability, resolution, and uncertainty.
- **Brier, G. W. (1950).** "Verification of Forecasts Expressed in Terms of Probability." *Monthly Weather Review*, 78(1), 1–3. The original Brier score.
- **DeGroot, M. H., & Fienberg, S. E. (1983).** "The Comparison and Evaluation of Forecasters." *The Statistician*, 32(1/2), 12–22. Foundational statement of calibration and refinement as separable properties of probabilistic forecasts.
- **Gneiting, T., & Raftery, A. E. (2007).** "Strictly Proper Scoring Rules, Prediction, and Estimation." *Journal of the American Statistical Association*, 102(477), 359–378. Modern definitive reference on proper scoring rules (Brier, log loss, and beyond).
- **Nixon, J., Dusenberry, M., Zhang, L., Jerfel, G., & Tran, D. (2019).** "Measuring Calibration in Deep Learning." *CVPR Workshops.* Introduces Adaptive Calibration Error and analyzes ECE's bin-count sensitivity. [arXiv:1904.01685](https://arxiv.org/abs/1904.01685).
- **Kumar, A., Liang, P., & Ma, T. (2019).** "Verified Uncertainty Calibration." *NeurIPS.* Debiased and kernel ECE estimators that correct ECE's finite-sample bias. [arXiv:1909.10155](https://arxiv.org/abs/1909.10155).
- **`sklearn.calibration.calibration_curve` and `CalibrationDisplay`.** Reference implementations for reliability diagrams with both `strategy="uniform"` and `strategy="quantile"` binning. [Docs](https://scikit-learn.org/stable/modules/generated/sklearn.calibration.calibration_curve.html).

## Recalibration methods (Chapter 3)

- **Platt, J. (1999).** "Probabilistic Outputs for Support Vector Machines and Comparisons to Regularized Likelihood Methods." *Advances in Large-Margin Classifiers.* MIT Press, pp. 61–74. The original Platt-scaling paper, including the label-Bayesian regularization detail. [PDF](https://www.cs.colorado.edu/~mozer/Teaching/syllabi/6622/papers/Platt1999.pdf).
- **Zadrozny, B., & Elkan, C. (2002).** "Transforming Classifier Scores into Accurate Multiclass Probability Estimates." *KDD.* Introduces isotonic regression as a calibrator and the one-vs-rest multiclass extension. [Paper (KDD)](https://dl.acm.org/doi/10.1145/775047.775151).
- **Niculescu-Mizil, A., & Caruana, R. (2005).** "Predicting Good Probabilities with Supervised Learning." *ICML.* The empirical comparison of Platt vs. isotonic across model families — the source of the "Platt for SVMs and boosted trees, isotonic for random forests with sufficient data" heuristic. [Paper (ICML)](https://www.cs.cornell.edu/~alexn/papers/calibration.icml05.crc.rev3.pdf).
- **Kull, M., Silva Filho, T., & Flach, P. (2017).** "Beta Calibration: A Well-Founded and Easily Implemented Improvement on Logistic Calibration for Binary Classifiers." *AISTATS.* Beta calibration as a family that generalizes Platt with better behavior at the tails. [Paper (PMLR)](https://proceedings.mlr.press/v54/kull17a.html).
- **Kull, M., Perello-Nieto, M., Kängsepp, M., Silva Filho, T., Song, H., & Flach, P. (2019).** "Beyond Temperature Scaling: Obtaining Well-Calibrated Multi-Class Probabilities with Dirichlet Calibration." *NeurIPS.* Extends the family to multiclass with Dirichlet-based calibrators. [arXiv:1910.12656](https://arxiv.org/abs/1910.12656).
- **`sklearn.calibration.CalibratedClassifierCV`.** Cross-validated Platt and isotonic calibration. [Docs](https://scikit-learn.org/stable/modules/generated/sklearn.calibration.CalibratedClassifierCV.html).
- **`sklearn.isotonic.IsotonicRegression`.** Reference PAVA implementation with `out_of_bounds` handling. [Docs](https://scikit-learn.org/stable/modules/generated/sklearn.isotonic.IsotonicRegression.html).

## Operating point selection and cost-sensitive learning (Chapter 4)

- **Elkan, C. (2001).** "The Foundations of Cost-Sensitive Learning." *IJCAI.* The canonical statement of the Bayes-optimal threshold as a function of the cost matrix and class prior. [PDF](https://cseweb.ucsd.edu/~elkan/rescale.pdf).
- **Neyman, J., & Pearson, E. S. (1933).** "On the Problem of the Most Efficient Tests of Statistical Hypotheses." *Philosophical Transactions of the Royal Society A*, 231, 289–337. The Neyman–Pearson lemma; the origin of the constrained-optimization framing for classification thresholds.
- **Provost, F., & Fawcett, T. (2001).** "Robust Classification for Imprecise Environments." *Machine Learning*, 42(3), 203–231. ROC convex hull and the cost-space framing for choosing operating points under uncertain costs. [Paper (Springer)](https://link.springer.com/article/10.1023/A:1007601015854).
- **Saito, T., & Rehmsmeier, M. (2015).** "The Precision-Recall Plot Is More Informative than the ROC Plot When Evaluating Binary Classifiers on Imbalanced Datasets." *PLoS ONE*, 10(3), e0118432. The reference paper for "use PR curves when the positive class is rare." [Paper (PLoS)](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0118432).
- **Davis, J., & Goadrich, M. (2006).** "The Relationship Between Precision-Recall and ROC Curves." *ICML.* Establishes the dominance relationship between PR and ROC and the interpolation subtlety for PR-AUC. [Paper (ACM)](https://dl.acm.org/doi/10.1145/1143844.1143874).
- **`sklearn.metrics.precision_recall_curve`, `roc_curve`, `det_curve`.** Reference implementations for the standard threshold-sweep curves. [Docs — model evaluation](https://scikit-learn.org/stable/modules/model_evaluation.html).

## Fairness metrics (Chapter 5)

- **Hardt, M., Price, E., & Srebro, N. (2016).** "Equality of Opportunity in Supervised Learning." *NeurIPS.* The equalized-odds and equal-opportunity definitions and the post-processing algorithm. [arXiv:1610.02413](https://arxiv.org/abs/1610.02413).
- **Dwork, C., Hardt, M., Pitassi, T., Reingold, O., & Zemel, R. (2012).** "Fairness Through Awareness." *ITCS.* The individual-fairness framework and the case against "fairness through unawareness" (i.e. removing the protected attribute from features). [arXiv:1104.3913](https://arxiv.org/abs/1104.3913).
- **Barocas, S., & Selbst, A. D. (2016).** "Big Data's Disparate Impact." *California Law Review*, 104(3), 671–732. The legal grounding for disparate-impact and disparate-treatment frameworks as they apply to ML. [Paper (SSRN)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2477899).
- **Barocas, S., Hardt, M., & Narayanan, A. (2019, ongoing).** *Fairness and Machine Learning: Limitations and Opportunities.* MIT Press (open web edition). The textbook that consolidates the definitions and their relationships. [fairmlbook.org](https://fairmlbook.org/).
- **Angwin, J., Larson, J., Mattu, S., & Kirchner, L. (2016).** "Machine Bias." *ProPublica.* The COMPAS reporting that motivated the impossibility papers. [ProPublica](https://www.propublica.org/article/machine-bias-risk-assessments-in-criminal-sentencing).
- **Dieterich, W., Mendoza, C., & Brennan, T. (2016).** "COMPAS Risk Scales: Demonstrating Accuracy Equity and Predictive Parity." Northpointe technical response. [Report (Equivant)](https://www.equivant.com/wp-content/uploads/ProPublica_Commentary_Final_070616.pdf).
- **EEOC Uniform Guidelines on Employee Selection Procedures (1978), 29 CFR §1607.4(D).** The four-fifths rule for adverse-impact screening in US employment contexts. [Regulation text (eCFR)](https://www.ecfr.gov/current/title-29/subtitle-B/chapter-XIV/part-1607).
- **Fiscella, K., & Fremont, A. M. (2006).** "Use of Geocoding and Surname Analysis to Estimate Race and Ethnicity." *Health Services Research*, 41(4p1), 1482–1500. The Bayesian Improved Surname Geocoding (BISG) methodology used to infer protected attributes when they are not directly collected. [Paper (Wiley)](https://onlinelibrary.wiley.com/doi/10.1111/j.1475-6773.2006.00551.x).
- **Buolamwini, J., & Gebru, T. (2018).** "Gender Shades: Intersectional Accuracy Disparities in Commercial Gender Classification." *FAccT.* The reference for intersectional analysis and the case that single-attribute audits miss the worst harms. [Paper (PMLR)](https://proceedings.mlr.press/v81/buolamwini18a.html).

## Fairness impossibility and tooling (Chapter 6)

- **Chouldechova, A. (2017).** "Fair Prediction with Disparate Impact: A Study of Bias in Recidivism Prediction Instruments." *Big Data*, 5(2), 153–163. The impossibility theorem for predictive parity, equal FPR, and equal FNR under differing base rates. [arXiv:1610.07524](https://arxiv.org/abs/1610.07524).
- **Kleinberg, J., Mullainathan, S., & Raghavan, M. (2017).** "Inherent Trade-Offs in the Fair Determination of Risk Scores." *ITCS.* The independent impossibility result for calibration within groups vs. balance for the positive/negative classes. [arXiv:1609.05807](https://arxiv.org/abs/1609.05807).
- **Pleiss, G., Raghavan, M., Wu, F., Kleinberg, J., & Weinberger, K. Q. (2017).** "On Fairness and Calibration." *NeurIPS.* Extends Kleinberg et al. and analyses the cost of enforcing calibration alongside other definitions. [arXiv:1709.02012](https://arxiv.org/abs/1709.02012).
- **Corbett-Davies, S., & Goel, S. (2018).** "The Measure and Mismeasure of Fairness: A Critical Review of Fair Machine Learning." *arXiv.* Survey of the definitions and the trade-offs a decision-maker actually faces. [arXiv:1808.00023](https://arxiv.org/abs/1808.00023).
- **Agarwal, A., Beygelzimer, A., Dudík, M., Langford, J., & Wallach, H. (2018).** "A Reductions Approach to Fair Classification." *ICML.* The `ExponentiatedGradient` and `GridSearch` reductions Fairlearn implements. [arXiv:1803.02453](https://arxiv.org/abs/1803.02453).
- **Kamiran, F., & Calders, T. (2012).** "Data Preprocessing Techniques for Classification without Discrimination." *Knowledge and Information Systems*, 33(1), 1–33. Foundational pre-processing mitigation. [Paper (Springer)](https://link.springer.com/article/10.1007/s10115-011-0463-8).
- **Fairlearn project.** Microsoft-maintained Python library for fairness measurement and mitigation. [fairlearn.org](https://fairlearn.org/) — see especially the [`MetricFrame` user guide](https://fairlearn.org/main/user_guide/assessment/metricframe.html) and the [`ThresholdOptimizer` docs](https://fairlearn.org/main/user_guide/mitigation/postprocessing.html).
- **Aequitas project.** University of Chicago DSaPP audit toolkit. [dssg.github.io/aequitas/](https://dssg.github.io/aequitas/) — see the [Group / Bias / Fairness modules](https://dssg.github.io/aequitas/api/aequitas.html).
- **AI Fairness 360 (AIF360).** IBM-originated fairness toolkit; broader mitigation menu than Fairlearn and useful as a cross-check. [aif360.readthedocs.io](https://aif360.readthedocs.io/).

## Regression and ranking evaluation (Chapter 7)

- **Hyndman, R. J., & Koehler, A. B. (2006).** "Another Look at Measures of Forecast Accuracy." *International Journal of Forecasting*, 22(4), 679–688. The reference statement of MAE / MAPE / sMAPE properties, the near-zero pathology, and the recommendation to use scaled errors. [Paper (Elsevier)](https://www.sciencedirect.com/science/article/pii/S0169207006000239).
- **Koenker, R., & Bassett, G. (1978).** "Regression Quantiles." *Econometrica*, 46(1), 33–50. The pinball / quantile loss and quantile regression.
- **Järvelin, K., & Kekäläinen, J. (2002).** "Cumulated Gain-Based Evaluation of IR Techniques." *ACM Transactions on Information Systems*, 20(4), 422–446. The original DCG / NDCG paper. [Paper (ACM)](https://dl.acm.org/doi/10.1145/582415.582418).
- **Wang, Y., Wang, L., Li, Y., He, D., Chen, W., & Liu, T.-Y. (2013).** "A Theoretical Analysis of NDCG Type Ranking Measures." *COLT.* Theoretical properties of NDCG and its consistency. [arXiv:1304.6480](https://arxiv.org/abs/1304.6480).
- **Voorhees, E. M. (1999).** "The TREC-8 Question Answering Track Report." *TREC.* The introduction of Mean Reciprocal Rank in the modern IR benchmarking context. [Paper (NIST)](https://trec.nist.gov/pubs/trec8/papers/qa_report.pdf).
- **Manning, C. D., Raghavan, P., & Schütze, H. (2008).** *Introduction to Information Retrieval.* Cambridge University Press. Chapter 8 covers MAP, MRR, NDCG, Precision@k, Recall@k with the standard formulations. [Open online edition](https://nlp.stanford.edu/IR-book/).
- **Joachims, T., Swaminathan, A., & Schnabel, T. (2017).** "Unbiased Learning-to-Rank with Biased Feedback." *WSDM.* Inverse-propensity-weighted evaluation for click-based ranking labels. [arXiv:1608.04468](https://arxiv.org/abs/1608.04468).
- **Chapelle, O., & Zhang, Y. (2009).** "A Dynamic Bayesian Network Click Model for Web Search Ranking." *WWW.* The reference DBN click model used for position-bias correction in offline ranking eval. [Paper (ACM)](https://dl.acm.org/doi/10.1145/1526709.1526711).
- **Zehlike, M., Bonchi, F., Castillo, C., Hajian, S., Megahed, M., & Baeza-Yates, R. (2017).** "FA*IR: A Fair Top-k Ranking Algorithm." *CIKM.* Entry point to fairness in ranking. [arXiv:1706.06368](https://arxiv.org/abs/1706.06368).
- **Diaz, F., Mitra, B., Ekstrand, M. D., Biega, A. J., & Carterette, B. (2020).** "Evaluation of Stochastic Rankings with Expected Exposure." *CIKM.* Exposure-based fairness metrics for ranking. [arXiv:2004.13157](https://arxiv.org/abs/2004.13157).
- **`sklearn.metrics.ndcg_score`, `average_precision_score`, `mean_absolute_error`, `mean_squared_error`, `r2_score`, `mean_absolute_percentage_error`.** Reference implementations. [Docs — model evaluation](https://scikit-learn.org/stable/modules/model_evaluation.html).
- **`pytrec_eval`.** Python bindings for the TREC `trec_eval` reference implementation. Use when your numbers must match published TREC baselines exactly. [Repository](https://github.com/cvangysel/pytrec_eval).

## Standards and legal frameworks referenced across the module

- **NIST AI Risk Management Framework (AI RMF 1.0), 2023.** The "Measure" function is the direct interface with evaluation engineering and includes fairness / calibration references. [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework).
- **EU AI Act (Regulation (EU) 2024/1689).** Annex III lists the high-risk categories where fairness auditing is a legal requirement, not an option. [EUR-Lex text](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).
- **EEOC Uniform Guidelines on Employee Selection Procedures.** 29 CFR Part 1607, the source of the four-fifths rule. [Regulation text (eCFR)](https://www.ecfr.gov/current/title-29/subtitle-B/chapter-XIV/part-1607).
- **CFPB — Home Mortgage Disclosure Act (HMDA) data.** The largest US public dataset with legally-mandated protected-attribute fields; used in fairness exercises. [ffiec.cfpb.gov/data-browser/](https://ffiec.cfpb.gov/data-browser/).
