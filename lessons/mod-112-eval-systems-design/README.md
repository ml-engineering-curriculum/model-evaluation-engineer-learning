# mod-112-eval-systems-design: Eval Systems Design and the Governance Interface

**Estimated effort:** 12 hours

## Learning objectives

- Translate a product specification into a release-gate eval plan with explicit pass thresholds and rollback criteria
- Compose a model card from eval evidence that satisfies internal review and external regulator expectations
- Map an eval plan to NIST AI RMF (Measure / Manage), ISO/IEC 25059 quality dimensions, and EU AI Act high-risk obligations where applicable
- Make the build-vs-buy decision across eval platforms (in-house, Arize Phoenix, Langfuse, Weave, OpenAI evals, vendor-hosted)
- Budget evaluations across cost / wall-clock / coverage and defend the trade-off to leadership

## Chapters

1. [The Governance Interface: What This Module Is About](01-the-governance-interface.md) — frames the module's altitude: evaluation as the seam where an engineering discipline meets product, regulator, leadership, and finance consumers; names the five governance artifacts the module builds (release-gate plan, model card, regulator crosswalk, platform decision dossier, budget defense) and draws the boundary with the doing-of-eval work in every earlier module.
2. [From Product Spec to Release-Gate Eval Plan](02-product-spec-to-release-gate-plan.md) — the translation discipline: intended-use derivation, gate categories (functional, safety, cost, latency, fairness), threshold derivation (baseline-relative vs. absolute vs. regulatory-committed), rollback criteria, plan-as-artifact structure, and the failure modes a good plan is designed to prevent.
3. [Model Cards Composed from Eval Evidence](03-model-cards-from-eval-evidence.md) — the model card as an eval-consumer artifact: the Mitchell et al. schema, evidence-to-section mapping, the internal-review vs. external-regulator dual audience, dataset / data-statement companions, versioning and update policy, and the anti-patterns (marketing card, ceremonial card, one-shot card) the discipline prevents.
4. [Mapping the Eval Plan to NIST AI RMF, ISO/IEC 25059, and the EU AI Act](04-regulatory-mapping-rmf-iso-eu-ai-act.md) — the crosswalk discipline: NIST AI RMF Measure / Manage functions and the Generative AI Profile, ISO/IEC 25059 quality characteristics, EU AI Act obligations for high-risk systems (Article 15 accuracy / robustness / cybersecurity, Annex IV documentation) and GPAI (Article 55), and the crosswalk-as-artifact structure.
5. [Build vs. Buy Across Eval Platforms](05-build-vs-buy-platform-matrix.md) — the decision framework across in-house, Arize Phoenix, Langfuse, W&B Weave, OpenAI evals, and vendor-hosted platforms: capability matrix, TCO modeling, data-residency and governance posture, exit / migration risk, and the staged-adoption pattern that keeps the decision reversible.
6. [Budgeting Evals Across Cost, Wall-Clock, and Coverage](06-budgeting-evals-cost-wallclock-coverage.md) — the three-axis trade-off: coverage-first vs. wall-clock-first vs. cost-first program shapes, the frontier and where organizations sit on it, the launch-cadence coupling, the leadership defense narrative, and the reallocation levers when the constraint changes.

## Structure

- `01-…md` … `06-…md`: lecture chapters (above).
- `exercises/`: per-exercise prompts. Solutions live in the paired `-solutions` repo.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
