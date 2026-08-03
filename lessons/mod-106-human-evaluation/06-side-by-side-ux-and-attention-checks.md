# Side-by-Side UX, Attention Checks, and Bot Detection

The interface an annotator uses is not neutral. Which side of the screen a response appears on, how long the labels stay visible, whether the annotator has a "same quality" option, whether they can go back and change an earlier answer — all of these shape the label distribution in ways that show up as bias in the aggregated numbers. This chapter is about the specific UX patterns that determine whether a human-eval interface produces trustworthy pairwise labels, and the two quality-control primitives — attention checks and bot detection — that filter the honest signal from the noise a paid crowd will inevitably contain.

The setting most of this chapter focuses on is the **pairwise side-by-side comparison**: annotator sees a prompt and two candidate responses (A and B), picks which is better (or declares a tie). This is Chatbot Arena's UX, MT-Bench's human study UX, and the shape most model-vs-model evaluations take. Almost everything here transfers to N-way ranking and to absolute-scoring interfaces with obvious modifications.

## The four decisions in a pairwise UX

**Presentation order.** If response A is always on the left and response B is always on the right, position bias in the annotator becomes indistinguishable from a real preference for whichever underlying system was tagged "A." The fix is randomization at *item level*: for each item, randomly assign each system to the left or right slot, and log which is which. Downstream, when computing win rates or Bradley-Terry ratings, aggregate over the underlying system identity, not the slot.

This is the human-side analog of the swap-and-average technique from mod-105 Chapter 4. On the model-judge side you'd run each item twice — once with each ordering — and average. Humans can't do that (they'd see the same item twice), so you randomize once per item and rely on the randomization averaging out across many items. Log the seed and the per-item assignment so a reviewer can verify.

**The verdict shape.** The three common options and what they cost:

1. **Binary (A better / B better).** Forces a preference on every item. Sensitive to noise on items that are genuinely a tie — the annotator picks arbitrarily and adds noise to the win rate. Highest per-item information when the systems really are distinguishable; worst when they are close.
2. **Ternary (A better / tie / B better).** Adds a genuine tie option. Better calibrated on close comparisons; the aggregate win rate reflects real preference more faithfully. The Chatbot Arena baseline shape.
3. **Graded (A much better / A slightly better / tie / B slightly better / B much better).** Adds magnitude. More per-item information; more annotator cognitive load; more noise on the boundaries between "slightly" and "much." Useful when downstream aggregation is Bradley-Terry-with-margin or a graded ELO variant.

For most production model-vs-model comparisons, ternary is the right default. Binary is fine when you know the systems are genuinely different in quality (a model-improvement A/B test where the treatment is expected to win). Graded is worth the load only when the downstream analysis actually consumes the magnitude.

**Whether the annotator sees the model names.** Do not show them. This is the human equivalent of a *blind* comparison, and it is not negotiable. Annotators shown "GPT-4o vs. Llama-3-8B" will label with their priors about the systems in play. Even model-family hints (a longer response, a specific formatting quirk) can leak identity — the aggregation layer needs to be robust to that leakage, but the UX should not amplify it. The Chatbot Arena crowdsourcing UX shows exactly this pattern: side-by-side responses, model names hidden until after the vote, then optionally revealed for user interest.

**Whether the annotator can revise.** Allow annotators to change their vote *before* submitting the item, but do not allow them to revise submitted votes. Post-submission revision creates a systematic bias where later, more-tired sessions overwrite earlier, fresher judgments — and it interacts badly with attention checks, which rely on the assumption that a submitted response is final. If an annotator wants to flag an item for review, that is a separate mechanism (a flag button, a note field), not a re-vote.

## The one paragraph on absolute-scoring UX

The four decisions map to absolute scoring with the following changes: presentation order is trivial (there is no A/B), the verdict shape is the anchored ordinal scale from Chapter 2, blinding is still important (do not reveal which system produced the response — the annotator is scoring the response, not the system), and revision policy is the same (revise-before-submit, no post-submit revision). The rest of this chapter — attention checks, bot detection, and quality-control primitives — applies unchanged.

## Attention checks: the primitive and its variants

An **attention check** is an item deliberately designed to have an unambiguous correct answer, injected into the annotator's queue with the same interface as a real item, whose purpose is to detect annotators who are labelling without engagement. It is a specific form of gold item (Chapter 5) with looser adjudication and higher injection rate.

Three common attention-check patterns.

**Explicit instruction check.** An item whose text or prompt includes an unambiguous instruction the annotator must follow to answer correctly. Common variants: an item that says "if you are reading this carefully, select `not_helpful`" embedded in the middle of a plausible prompt; a pairwise item where the prompt explicitly asks the annotator to prefer the shorter response. These are effective but obvious once you have seen a few — sophisticated crowd workers pattern-match on them. Use sparingly and vary the phrasing.

**Trivial correctness check.** An item where one response is obviously and dramatically better than the other — a pairwise comparison where response B is a paragraph of gibberish or a copy of the prompt, or an absolute-scoring item where the response is clearly nonsensical. If an annotator picks the gibberish, they are not reading. These are less pattern-matchable but require more design work per item.

**Duplicate consistency check.** The same item is shown twice in an annotator's session (with different item IDs, ideally many items apart) and the labels are compared. An annotator whose labels on the same item flip between sessions is either drifting or clicking randomly. This is the most subtle and hardest to game; it also produces useful signal about within-annotator consistency.

**Rate.** 5–10% of the annotator's queue is a common target. Higher rates are more sensitive to inattentive annotators but eat into productive labelling. Lower rates delay detection.

**Threshold.** A per-annotator threshold on attention-check accuracy — 90% is a common baseline. An annotator dropping below the threshold is paused pending review, not immediately banned; borderline cases are often training issues that a quick retraining session resolves.

## Bot detection

Bot detection is the harder cousin of attention checking. It targets not inattentive humans but *scripted or LLM-mediated* labelling. Some crowd workers now use LLMs to generate labels (Veselovsky et al. 2023 documented this on Prolific for text-summarization tasks, with 33–46% of crowd workers estimated to use LLMs). The detection primitives are:

**Time-per-item bounds.** Every labelling interface should record per-item time. A distribution with an unusually large spike at very low times (< 3 seconds on a task that takes 20+ seconds of reading) is a signal. Very *high* times can also be suspicious, though they are more commonly slow-but-honest annotators. The response to a low-time spike is per-annotator review, not automated banning — measurement times are noisy on real work.

**Response entropy.** On tasks that produce free-text notes or free-form justifications alongside labels, the *distribution* of those notes across an annotator's session is informative. An annotator writing near-identical notes on visually different items — or notes with characteristic LLM phrasing ("as an AI language model," "in summary") — is a signal. Absent free-text output, this primitive is not available; adding a required "one-sentence reason" field is one way to enable it.

**Behavioral consistency checks.** Cursor movement, click position within the label button, time between reading and clicking. These are harder to collect and privacy-sensitive; only some labelling tools expose them; typically deployed only in high-stakes or high-fraud settings.

**Reputation on the platform.** All major crowd platforms (Prolific, Scale, Surge, Mercor) publish per-worker completion rates and quality scores from prior tasks. Filtering the pool to workers with high completion rates and no recent quality flags is the cheapest bot filter, and the one most projects should be running by default.

Bot detection is not a solved problem — the arms race with LLM-mediated labelling is active. The pragmatic stance is: (1) know what fraction of your task's labels are at risk of being LLM-generated (a summarization task on Prolific is high-risk; a code annotation task requiring a specific IDE is low-risk); (2) design attention checks that would fail on naive LLM completion; (3) treat unusually low variance in a worker's label distribution as suspicious; (4) monitor time-per-item; (5) prefer in-house or vetted-expert annotators for tasks where bot-labelling would materially bias the eval.

## Where the UX and the QC primitives meet the guideline

Every UX and QC primitive in this chapter is downstream of the annotation guideline (Chapter 2). Concretely:

- Randomized presentation order requires a guideline that is *response-focused* (Rule 1) rather than model-focused; if the guideline references "the model," annotators will look for identity cues in the response.
- Ternary verdict shape requires the guideline to define what a "tie" actually means for the task. "Tie" can mean "same quality" or "genuinely uncertain" or "both are so bad the difference doesn't matter"; each has different downstream implications.
- Attention checks need to be labellable under the *same* guideline as real items. An attention check whose correct answer requires reading a hidden instruction violates the guideline's rules; the annotator failing it is following the guideline correctly. Design attention checks to be edge cases the guideline unambiguously handles, not violations of it.
- Bot detection via free-text reason fields requires the guideline to specify what a reason should look like ("one sentence, referring to the specific content of the response").

The UX and the guideline are not two artifacts; they are one specification with two surfaces. Changes to either invalidate calibration on the other. When you version the guideline (Chapter 2, Rule 7), you are versioning the UX contract as well.

## Common failure modes

**Position not randomized, aggregation forgets to unrandomize.** Even when items are randomized on presentation, the downstream aggregation must know which underlying system was on which side per item. If your logs record only "annotator picked left" without recording which system was on the left for that item, you have thrown away the ability to compute an unbiased win rate. Log both, always.

**Attention checks at 30% of the queue.** Annotators notice, complain about repetitive-feeling items, and their productivity drops. Bring the rate back to 5–10%.

**Attention checks whose "correct" answer is arguably wrong.** An attention check that says "select `helpful`" but where the actual reply is genuinely partial gets counted as a failure for careful annotators. Design attention checks with an unambiguous *content-level* answer, not just an instruction the annotator is expected to follow blindly.

**Bot detection via bans without appeal.** An annotator temporarily flagged by a noisy metric loses income and access to the platform; the pool shrinks; other honest annotators worry. Every bot-detection or attention-check-failure signal should route to a review queue, not to an automated ban. The false-positive cost of a wrong ban is high in both platform reputation and pool retention.

**Revising submitted votes.** The interface allows an annotator to change a submitted vote if they realize they misread the item. This looks like a quality-of-life feature; it corrupts the attention-check signal (an annotator who "changed their mind" on an obvious attention check might be revising because they got flagged mid-session) and produces asymmetric data (recent items get revised more often than distant items). Disallow post-submission revision.

## Summary

A pairwise side-by-side UX has four load-bearing decisions — per-item randomized presentation, ternary verdict shape (as a good default), blinding of model identity, and no post-submission revision — each of which corresponds to a specific bias mode that the aggregation math cannot fix downstream. Attention checks (explicit-instruction, trivial-correctness, and duplicate-consistency) run at 5–10% of the queue and gate an annotator's status against a per-annotator accuracy threshold. Bot detection stacks time-per-item, response entropy, behavioral primitives, and platform reputation, with routing to review rather than automated bans; the ecosystem is in an active arms race with LLM-mediated labelling and every project should know its risk exposure. All of these are downstream of the annotation guideline and the UX contract is versioned with the guideline. The final chapter of the module confronts the sourcing question: who is on the other end of that UX — a crowd platform, an in-house team, an expert panel — and what the build-vs-buy tradeoff actually looks like.
