# Resources for mod-112-eval-systems-design (Eval Systems Design and the Governance Interface)

Primary references for the material in this module. Prefer the original paper, the maintained tool repository, or the official regulatory text over secondary summaries. Chapter 4's discipline is explicit: cite by version.

## Model cards, data statements, and datasheets (Chapter 3)

- **Mitchell, M., Wu, S., Zaldivar, A., Barnes, P., Vasserman, L., Hutchinson, B., Spitzer, E., Raji, I. D., & Gebru, T. (2019).** *Model Cards for Model Reporting.* Proceedings of FAT* '19. The founding paper for the nine-section model-card schema this chapter builds on. [arXiv:1810.03993](https://arxiv.org/abs/1810.03993) · [ACM DL](https://dl.acm.org/doi/10.1145/3287560.3287596).
- **Bender, E. M., & Friedman, B. (2018).** *Data Statements for Natural Language Processing: Toward Mitigating System Bias and Enabling Better Science.* Transactions of the ACL, 6, 587–604. The reference for the data-statement companion to the model card. [MIT Press](https://direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00041/43452).
- **Gebru, T., Morgenstern, J., Vecchione, B., Vaughan, J. W., Wallach, H., Daumé III, H., & Crawford, K. (2021).** *Datasheets for Datasets.* Communications of the ACM, 64(12), 86–92. The alternative datasheet schema for dataset documentation. [arXiv:1803.09010](https://arxiv.org/abs/1803.09010) · [ACM DL](https://dl.acm.org/doi/10.1145/3458723).
- **Hugging Face model cards.** The de-facto model-card ecosystem for the open-model community: [huggingface.co/docs/hub/model-cards](https://huggingface.co/docs/hub/model-cards). The Hub's card format has become the practical reference for public-facing model-card publication.
- **Partnership on AI — About ML.** Guidance on documentation for ML models, including model-card templates and case studies: [partnershiponai.org/workstream/about-ml](https://partnershiponai.org/workstream/about-ml/).

## Standards and regulatory context (Chapters 4, and referenced throughout)

- **NIST AI Risk Management Framework (AI RMF 1.0), 2023.** The reference framework whose Measure and Manage functions the module's crosswalk targets. [NIST landing page](https://www.nist.gov/itl/ai-risk-management-framework) · [PDF](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf).
- **NIST AI RMF Generative AI Profile (NIST AI 600-1), 2024.** The GAI-specific extension of the RMF with concrete measurement expectations for generative systems and a taxonomy of GAI-specific risks. [DOI](https://doi.org/10.6028/NIST.AI.600-1).
- **NIST AI 100-2 Adversarial Machine Learning taxonomy (2023).** The taxonomy of adversarial ML techniques the RMF's security clauses reference. [NIST AI 100-2](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2023.pdf).
- **EU AI Act (Regulation (EU) 2024/1689).** The binding EU regulation. Article 15 (accuracy, robustness, cybersecurity), Annex IV (technical documentation), Article 55 (GPAI with systemic risk), and Article 72 (post-market monitoring) are the load-bearing clauses for eval-program crosswalks. [EUR-Lex text](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).
- **EU AI Office (European Commission).** The office responsible for implementing and enforcing the AI Act. Announcements, delegated acts, and codes of practice land here: [digital-strategy.ec.europa.eu/en/policies/ai-office](https://digital-strategy.ec.europa.eu/en/policies/ai-office).
- **ISO/IEC 42001:2023, Information technology — Artificial intelligence — Management system.** The AI-specific management-system standard whose Clause 9 (performance evaluation) and Clause 10 (improvement) pair with this module's discipline. [ISO catalogue entry](https://www.iso.org/standard/81230.html).
- **ISO/IEC 25059:2023, Software engineering — SQuaRE — Quality model for AI systems.** The AI-specific quality model that Chapter 4's crosswalk targets. [ISO catalogue entry](https://www.iso.org/standard/80655.html).
- **ISO/IEC 25010:2011, Systems and software engineering — SQuaRE — System and software quality models.** The base SQuaRE quality model that 25059 extends. [ISO catalogue entry](https://www.iso.org/standard/35733.html).
- **ISO/IEC 23894:2023, Information technology — Artificial intelligence — Guidance on risk management.** The AI-specific extension of ISO 31000 risk-management guidance. [ISO catalogue entry](https://www.iso.org/standard/77304.html).
- **ISO/IEC 5259 series (data quality for analytics and ML).** Standards on data-quality frameworks and measures for ML datasets — useful for the data-statement companion. [ISO 5259-1](https://www.iso.org/standard/81088.html).
- **U.K. AI Safety Institute — Inspect and evaluations reports.** Public evaluation reports from AISI whose format is a useful reference for external safety-evaluation disclosure. [aisi.gov.uk](https://www.aisi.gov.uk/).
- **U.S. Executive Order 14110 (superseded 2025) and successor Executive Order guidance.** U.S. executive-branch AI policy is a moving target; consult the current administration's operative order for U.S. federal-contracting evaluation expectations. <!-- needs-research: identify the operative U.S. federal executive order or OMB memorandum on AI evaluation as of the exercise date; the landscape shifted after EO 14110 was revoked. -->

## Eval platforms and observability tools (Chapter 5)

- **Arize `Phoenix`.** Open-source LLM observability and evaluation. Docs: [docs.arize.com/phoenix](https://docs.arize.com/phoenix). Repository: [`Arize-ai/phoenix`](https://github.com/Arize-ai/phoenix).
- **Arize AX.** Arize's hosted platform for LLM evaluation and observability at production scale: [arize.com/ax](https://arize.com/ax/).
- **Langfuse.** Open-source LLM observability with prompt management and eval scoring. Docs: [langfuse.com/docs](https://langfuse.com/docs). Repository: [`langfuse/langfuse`](https://github.com/langfuse/langfuse). Self-hosted or Langfuse Cloud.
- **Weights & Biases Weave.** W&B's LLM observability layer with `@weave.op` decorator model and integration with the W&B experiment-tracking substrate. Docs: [weave-docs.wandb.ai](https://weave-docs.wandb.ai/).
- **OpenAI evals.** OpenAI's open-source evaluation harness and task catalog. Repository: [`openai/evals`](https://github.com/openai/evals). Not a full platform — a runner and task collection.
- **LangSmith (LangChain).** LangChain's hosted observability and eval product. Docs: [docs.smith.langchain.com](https://docs.smith.langchain.com/).
- **Braintrust.** Hosted eval platform with strong dataset-iteration UX. Docs: [braintrust.dev/docs](https://www.braintrust.dev/docs).
- **Patronus AI.** Hosted eval platform with model-graded evaluators and a research-backed judge suite. Docs: [docs.patronus.ai](https://docs.patronus.ai/).
- **Humanloop.** Hosted eval and prompt-management platform: [humanloop.com](https://humanloop.com/).
- **Confident AI (DeepEval).** Open-source eval framework (`deepeval`) with a hosted evaluation platform. Repository: [`confident-ai/deepeval`](https://github.com/confident-ai/deepeval).
- **Galileo AI.** Hosted eval and observability platform for LLM applications: [galileo.ai](https://galileo.ai/).
- **Fiddler AI.** ML-observability platform with LLM-specific evaluation surfaces: [fiddler.ai](https://www.fiddler.ai/).
- **Comet Opik.** Comet's open-source LLM eval and observability offering: [github.com/comet-ml/opik](https://github.com/comet-ml/opik).
- **`Portkey`.** Model-gateway product with routing, retries, and eval-adjacent observability: [portkey.ai](https://portkey.ai/).
- **`LiteLLM`.** Unified vendor client for hosted models — the substrate many eval platforms use for their vendor-abstraction layer: [github.com/BerriAI/litellm](https://github.com/BerriAI/litellm).

Category note: the hosted-eval-platform space moves quarter over quarter. The list above is representative of the vendors most eval programs will consider in 2026; specific feature parity should be checked against each vendor's current documentation before making a commitment. <!-- needs-research: refresh the vendor list before any external publication of a build-vs-buy dossier that cites this module. -->

## SLOs, budgets, and release engineering (Chapters 2, 6)

- **Beyer, B., Jones, C., Petoff, J., & Murphy, N. R. (2016).** *Site Reliability Engineering.* O'Reilly. The canonical text on SLOs, error budgets, and the release-engineering discipline the release-gate plan sits inside. Full text online: [sre.google/sre-book/table-of-contents](https://sre.google/sre-book/table-of-contents/).
- **Beyer, B., Murphy, N. R., Rensin, D. K., Kawahara, K., & Thorne, S. (Eds.) (2018).** *The Site Reliability Workbook.* O'Reilly. The applied companion; Chapters 2–4 walk SLO derivation from historical distributions. [sre.google/workbook/table-of-contents](https://sre.google/workbook/table-of-contents/).
- **Google Cloud SRE resource: "Error budget policy."** Reference for the error-budget-policy discipline that compose with the release-gate plan's override authority. [sre.google/workbook/error-budget-policy](https://sre.google/workbook/error-budget-policy/).
- **Google's launch-review discipline (LCE — Launch Coordination Engineering).** Chapter 27 of *Site Reliability Engineering* — the reference for launch-gate discipline at scale.

## Vendor-batch APIs, prompt caching, and cost-optimization primitives (Chapter 6)

- **Anthropic Message Batches API.** 24-hour SLA batch pricing with ~50% discount: [docs.claude.com/en/docs/build-with-claude/batch-processing](https://docs.claude.com/en/docs/build-with-claude/batch-processing).
- **OpenAI Batch API.** 24-hour SLA batch endpoint: [platform.openai.com/docs/guides/batch](https://platform.openai.com/docs/guides/batch).
- **Anthropic prompt caching.** [docs.claude.com/en/docs/build-with-claude/prompt-caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching).
- **OpenAI prompt caching.** [platform.openai.com/docs/guides/prompt-caching](https://platform.openai.com/docs/guides/prompt-caching).
- **Google Gemini context caching.** [ai.google.dev/gemini-api/docs/caching](https://ai.google.dev/gemini-api/docs/caching).
- **FinOps Foundation — cloud FinOps discipline.** The financial-operations discipline the eval-program budget defense composes with when the organization has an existing FinOps practice: [finops.org](https://www.finops.org/).

## Governance frameworks and complementary standards

- **OECD AI Principles (2019, updated).** The intergovernmental principles that inform many national AI policies. [oecd.ai/en/ai-principles](https://oecd.ai/en/ai-principles).
- **Council of Europe Framework Convention on Artificial Intelligence and Human Rights, Democracy and the Rule of Law (2024).** The Council of Europe's binding AI treaty for signatory states. [Council of Europe treaty page](https://www.coe.int/en/web/artificial-intelligence/the-framework-convention-on-artificial-intelligence).
- **Singapore Model AI Governance Framework and AI Verify Foundation.** Non-binding governance frameworks with practical evaluation guidance. [aiverifyfoundation.sg](https://aiverifyfoundation.sg/).
- **U.K. Data Ethics Framework and Algorithmic Transparency Recording Standard.** U.K.-government-facing transparency artifacts. [gov.uk data ethics guidance](https://www.gov.uk/government/publications/data-ethics-framework).

## Foundational papers and reports for the eval-governance interface

- **Raji, I. D., Smart, A., White, R. N., Mitchell, M., Gebru, T., Hutchinson, B., Smith-Loud, J., Theron, D., & Barnes, P. (2020).** *Closing the AI Accountability Gap: Defining an End-to-End Framework for Internal Algorithmic Auditing.* FAT* '20. The reference for the internal-audit discipline the module's crosswalks compose into. [arXiv:2001.00973](https://arxiv.org/abs/2001.00973).
- **Weidinger, L., et al. (DeepMind, 2021 and follow-ups).** *Ethical and social risks of harm from Language Models* (and subsequent taxonomy papers). Reference for the risk taxonomy the safety-gate crosswalk composes with. [arXiv:2112.04359](https://arxiv.org/abs/2112.04359).
- **Bommasani, R., et al. (Stanford CRFM, 2021).** *On the Opportunities and Risks of Foundation Models.* A broad framing paper for the foundation-model governance context. [arXiv:2108.07258](https://arxiv.org/abs/2108.07258).
- **Solaiman, I., et al. (2023).** *Evaluating the Social Impact of Generative AI Systems in Systems and Society.* [arXiv:2306.05949](https://arxiv.org/abs/2306.05949). Useful frame for the categories of social-impact evaluation the model card should acknowledge.
- **Anthropic Responsible Scaling Policy (RSP).** Public policy document specifying evaluation-triggered capability thresholds and the response protocol. Useful reference for the "eval outputs drive governance decisions" discipline the module is written around. [anthropic.com/rsp](https://www.anthropic.com/rsp).
- **OpenAI Preparedness Framework.** The analogous OpenAI document, structured around risk-category thresholds and corresponding safeguards. [openai.com/preparedness](https://openai.com/index/openai-preparedness-framework/).

## Adjacent module cross-references

- **mod-101 (evaluation foundations).** Confidence-interval discipline (Chapter 4) that the release-gate plan's threshold derivation depends on; multiple-comparisons discipline (Chapter 6) that the plan's gate-budget uses.
- **mod-102 (benchmark engineering).** License, provenance, contamination, and gold-set-with-IAA disciplines that the data-statement companion compiles.
- **mod-103 (classical ML eval depth).** Per-slice metrics discipline (Chapter 1) and fairness measurement (Chapter 4) that the model card's factors and quantitative-analyses sections compile.
- **mod-104 (LLM benchmark harnesses).** The runner substrate whose task catalogs the eval program uses.
- **mod-105 (LLM-as-judge platforms).** Judge calibration and agreement statistics that the plan's judge-tier and the card's metric-justification sections depend on.
- **mod-106 (human evaluation).** Human-panel discipline the plan's safety-gate and the card's ethical-considerations sections may reference.
- **mod-107 (generative and multimodal eval).** Multimodal-specific measurement whose coverage the model card's factors section discloses.
- **mod-108 (agent and tool eval).** Agent-specific measurement that composes with the plan's functional-quality and safety sections.
- **mod-109 (safety and red-team eval).** The safety-measurement substrate the plan's safety gates and the card's ethical-considerations section compile.
- **mod-110 (production eval and regression).** The offline / shadow / A/B / continuous-monitoring altitudes that the plan's gates and rollback criteria sit inside.
- **mod-111 (eval platform engineering).** The registry (Chapter 2) and warehouse (Chapter 5) substrate that every artifact in this module cites; the platform's own SLOs (Chapter 6) compose with the eval-program SLOs the release-gate plan implies.
