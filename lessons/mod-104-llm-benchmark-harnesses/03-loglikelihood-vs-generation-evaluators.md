# Log-Likelihood vs. Generation Evaluators

The most consequential design choice in any LLM benchmark is *how you extract the model's answer*. For a multiple-choice question you have two options: score each candidate answer's log-likelihood under the model and pick the argmax (the "log-likelihood" or "ranking" evaluator), or ask the model to *generate* an answer as text and parse the result (the "generation" evaluator). The two evaluators, applied to the same model on the same items, routinely disagree by 5–15 accuracy points, and neither one is trivially right. This chapter is about what each evaluator actually measures, why the gap is systematic, and how to pick between them (or run both) in a defensible way.

## The two request types

Every LLM benchmark harness in this module supports both request patterns even when the surface vocabulary hides the choice.

**Log-likelihood.** Given a context `C` and a candidate continuation `T`, the model returns `log P(T | C) = Σᵢ log P(tᵢ | C, t<ᵢ)`. No sampling, no stopping condition, deterministic given the model weights. For a multiple-choice item with choices `T₁ ... Tₖ`, the evaluator computes each `log P(Tⱼ | C)` and picks `argmaxⱼ`. This is what lm-eval's `output_type: multiple_choice` does; it is what HELM calls the "separate" adapter; it is `Model.score()` in Inspect (rarely used).

**Generation.** Given a context `C`, the model samples tokens `t₁, t₂, ...` (deterministically if `do_sample=false` / `temperature=0`) until a stop condition. The evaluator parses the generated string to extract an answer. For a multiple-choice item the prompt usually reads "Answer with A, B, C, or D." and the parser extracts the letter. This is `output_type: generate_until` in lm-eval, the "multiple_choice_joint" adapter in HELM, `generate() + choice()` scorer in Inspect, and the default in OpenAI evals.

The log-likelihood evaluator asks: **which candidate did the model find most plausible?** The generation evaluator asks: **what does the model actually write down?** These are different questions, and a model can be very good at one and very bad at the other.

## Why the two evaluators disagree

Four mechanisms drive the log-likelihood vs. generation gap. In real benchmarks, all four are usually at play.

**Chat / instruction tuning distortion.** A pre-trained base model, given `"Question: X\nAnswer:"`, will freely assign log-likelihood mass to a completion like `" 42."` — even if that answer is wrong — because base models model the training corpus as text. A chat-tuned model, given the same context, may want to *say* `"Sure! The answer is 42."` because that is what its instruction tuning taught it to do. Log-likelihood ranking of bare answer strings against a chat-tuned model punishes the model for its own instruction tuning: `log P(" A" | context)` is small because the model prefers to open with `"The answer is A"`. Generation, then extract-a-letter, avoids this by letting the model write in its natural style. The 2023 paper *"Language Models Understand Numbers, at Least Partially"* and follow-up work document this systematically: chat-tuned model scores on MMLU can move by 5–10 points depending on whether you rank or generate.

**Answer-length bias in the log-likelihood.** `log P(T | C)` is a sum over tokens; longer candidates accumulate more (negative) log-prob. If the choices in a task are uneven in length — one is `"yes"` and another is `"probably not, based on the passage"` — the shorter choice will win the argmax under the model whose actual preference between the two propositions is neutral. This is why `acc_norm` exists: normalize by the number of tokens (or bytes) in the candidate. `acc` uses the raw log-likelihood argmax; `acc_norm` divides each candidate's log-prob by its byte length. The two disagree by 1–5 points on the same task, and *which one to prefer* is not obvious — length-normalization removes a spurious bias but introduces its own (short answers get systematically boosted). The HellaSwag task and the ARC tasks were early public demonstrations of this trade-off.

**Prompt-template asymmetry.** The log-likelihood evaluator only rewards the *first* token(s) of the candidate that discriminate it from the others; the tokens shared with other candidates carry equal probability under all hypotheses and wash out. If your prompt template does not put the answer discriminator right at the boundary, the log-likelihood signal is weakened. Small changes in prompt phrasing — `"Answer:"` vs. `"The answer is:"` — change where the discriminative tokens land and can move ranking accuracy by several points. Generation is less sensitive to this because the model has more room to reach the answer in its own way, but it introduces its own template dependence (see Chapter 7).

**Decoding and parsing noise.** Generation evaluators need to parse the model's output. If the model writes `"I think the answer is (B) because the passage says..."` and your extractor looks for the pattern `^([A-D])\.`, you get zero for a correct answer. Log-likelihood ranking has no parser, so no parser bug. On the other hand, generation evaluators can be softened with model-graded rubrics or better filters, whereas log-likelihood errors due to instruction tuning are structural.

## When each evaluator is the right choice

There is no universal answer; there is a defensible mapping from *what you are trying to measure* to *which evaluator captures it*.

**Prefer log-likelihood ranking when:**

- You are evaluating a **base model** or a completion-style model that has not been chat-tuned. The instruction-tuning distortion does not apply.
- The task has **fixed, short candidates** of comparable length (e.g. ARC, HellaSwag's four endings, TruthfulQA-MC1, BoolQ). Length asymmetry is manageable and normalization is well-understood.
- You need **maximum sample efficiency**. Log-likelihood is deterministic and single-request; generation is often multi-token and (with sampling) non-deterministic. For very expensive models, log-likelihood saves cost.
- You want to compare **base models** across a pretraining regime. Log-likelihood is closer to the pretraining objective and less confounded by post-training.

**Prefer generation when:**

- You are evaluating a **chat-tuned or instruction-tuned model** and want the number to reflect how it behaves in its intended deployment.
- The **task has open-ended answers** (math word problems, code generation, summarization, tool calling). Log-likelihood over a target string is a bad proxy for "did the model actually produce a working solution."
- The **candidate space is not enumerable** (free-form answers, long text). Log-likelihood requires you to enumerate candidates; generation does not.
- You need to run the same eval **through an API that does not expose log-probs**. Most commercial APIs restrict or do not expose token-level log-probs over supplied continuations; you cannot run a log-likelihood eval without that capability.

**Run both when:**

- You are launching a **novel model or a novel task** and want to characterize the gap before picking a headline number. Any launch report that quotes a single MMLU number without saying which mode is a partial report.
- You are **reproducing a published number** and are uncertain which mode the original authors used. The gap between the two modes on your rerun will usually tell you.
- The **reader will make a decision from the score** (ship / hold, upgrade / rollback) and the two evaluators would flip the decision. This should be surfaced, not hidden.

## Normalisation: the small print inside log-likelihood

For log-likelihood-ranked tasks, "which normalisation" is a real choice, not a footnote.

**Raw log-likelihood (`acc`).** `log P(T | C)`. Length-biased toward shorter candidates.

**Length-normalised (`acc_norm`).** `log P(T | C) / len(T)`, where `len` is by default in bytes (not tokens — bytes are tokenizer-invariant, which matters when comparing across models with different tokenizers). This is what lm-eval's `acc_norm` reports. Reduces length bias but *inverts* the bias for very short candidates.

**Token-normalised.** Same as byte-normalised but divides by token count. Discouraged for cross-tokenizer comparison because a tokenizer that fragments a candidate more will lower its per-token score. Rarely defensible.

**PMI-normalised.** `log P(T | C) - log P(T | Ø)`, where `Ø` is a null context (or a task-generic context). Removes the prior probability of the candidate string; a candidate that is a common phrase does not get an advantage over an unusual one. Popularized by the *"Surface Form Competition"* paper (Holtzman et al. 2021) as a fix for base-model MC scoring. Requires two log-likelihood requests per candidate, doubling the eval cost.

**Choice-only likelihood.** `log P(T | context that mentions all choices and asks for one)`. In HellaSwag-style tasks the choices are so long that a special "conditional on all choices" formulation avoids the raw prior swamping the discriminative signal. Task-specific; document what you are computing.

Which normalisation to use is a per-task decision. For MMLU the widely-cited numbers are `acc` under a specific prompt template; for HellaSwag they are `acc_norm`; for TruthfulQA-MC1 they are `acc`. Chapter 4 on reproduction is largely about knowing which normalisation each published number is under.

## Two worked comparisons

**Case 1: Llama-3-8B-Instruct on MMLU.** With log-likelihood ranking under the classic MMLU prompt (`"The following are multiple choice questions ... Question: {q}\nA. {a}\nB. {b}\nC. {c}\nD. {d}\nAnswer:"`) and 5-shot, you get one number. With generation ("respond with only the letter of the correct answer") the number is typically several points different. Both are defensible; both should be reported. The model card's headline number will be one of them; do not compare against the wrong one.

**Case 2: A code-completion task.** "Given this function signature and docstring, write the body." Log-likelihood ranking is not applicable — the target is open-ended. Generation is the only sane evaluator. But generation requires a metric (exact match on the target body? unit-test pass rate? bleu?), and the choice of metric is where the interesting work is. Log-likelihood on the *reference solution* under the model is sometimes reported as a proxy but is a bad one — the model can assign high log-likelihood to a solution it would never generate under any decoding strategy.

## Running both in lm-eval-harness

Follow Chapter 2's `customer_sentiment` example. Register two tasks with the same dataset:

- `customer_sentiment_mc` with `output_type: multiple_choice` — ranks over `["positive", "neutral", "negative"]`.
- `customer_sentiment_gen` with `output_type: generate_until` — parses the model's free-form answer through a `regex` filter.

Run both:

```bash
lm_eval \
  --model hf --model_args pretrained=<model>,dtype=bfloat16 \
  --tasks customer_sentiment_mc,customer_sentiment_gen \
  --num_fewshot 4 --apply_chat_template --fewshot_as_multiturn \
  --output_path runs/customer_sentiment --log_samples
```

Then join the two `--log_samples` JSONL files on `doc_id` and compute the per-item disagreement rate. A per-item confusion matrix — "log-likelihood-ranking correct, generation correct" vs. the other three cells — is more informative than the two headline numbers because it tells you *where* the two evaluators disagree.

Typical patterns you will see:

- **Systematic disagreement on a subset of item types.** The log-likelihood ranker is wrong on items where the correct label's string is a common word (positive-prior-driven mistake). The generation evaluator is wrong when the model writes a hedged response that the parser fails on.
- **Chat-tuned model wins under generation, loses under log-likelihood.** The archetypal case. Report both.
- **Base model wins under log-likelihood, loses under generation.** Also common — base models rank fine but generate off-task.

## Reporting the gap

If you report a single number, you should be able to defend the choice of evaluator in one sentence. If you report both, do so in a table:

| Evaluator | Metric | Score (95% CI) | n | Notes |
| --- | --- | --- | --- | --- |
| log-likelihood ranking | `acc_norm` | 0.712 (0.688, 0.735) | 1000 | byte-normalised, prompt v3 |
| generation → regex extract | `exact_match` | 0.663 (0.639, 0.687) | 1000 | greedy, `T=0`, regex `(positive|neutral|negative)` |
| **agreement rate** | — | 0.821 | 1000 | of items where both scored, fraction where the two agreed on the label |

The agreement rate is the number a launch reviewer will actually ask about, because it tells them how sensitive the headline is to the evaluator choice. If the two evaluators agree on 95%+ of items, the choice is aesthetics. If they agree on 70%, the choice is the eval.

## Summary

Log-likelihood and generation evaluators ask different questions of a model — "what did you find most plausible?" versus "what did you write down?" — and routinely disagree by several accuracy points on the same task and the same model. The gap is driven by chat / instruction tuning, answer-length bias in raw log-likelihood, prompt-template asymmetry, and parser noise. Normalisation choices (`acc`, `acc_norm`, PMI) further split the log-likelihood family. Prefer log-likelihood for base models and short-candidate MC tasks; prefer generation for chat models and open-ended answers; run both when launching a novel model, reproducing a public number, or when the decision the reader will make depends on the choice. Report the per-item agreement rate, not just the two headline numbers. Chapter 4 turns to the harness-level cousin of this problem: reproducing a *specific* published number, where the evaluator choice is only one of four gap sources you will actually see.
