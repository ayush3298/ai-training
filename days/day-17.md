# Day 17 — Evaluation, part 3 — online eval, hallucination & the feedback loop

> [← Day 16](day-16.md) · [All days](README.md) · [Day 18 →](day-18.md)

**Today's chapter:** [Chapter 7 — Evaluation, Safety & Reliability: shipping with confidence, not vibes](../chapters/07-evaluation.md)  
**Sections:** Parts G–J  
**Time:** ~3 hrs

## Goal
Evaluate without a gold answer in production, name and detect hallucination (and reframe BLEU/ROUGE), and turn user signals into eval cases.

## Why today matters
Offline eval runs out at the prod boundary. Online eval + a feedback loop is how the system keeps improving after you ship.

## Study
Work through these sections:

- [Part G — Online eval & evaluating without a gold answer](../chapters/07-evaluation.md#part-g--online-eval--evaluating-without-a-gold-answer)
- [Part H — Hallucination: a named taxonomy, detection, and the BLEU/ROUGE reframe](../chapters/07-evaluation.md#part-h--hallucination-a-named-taxonomy-detection-and-the-bleurouge-reframe)
- [Part I — Testing code vs. evaluating the model](../chapters/07-evaluation.md#part-i--testing-code-vs-evaluating-the-model)
- [Part J — The feedback loop: turning user signals into eval cases](../chapters/07-evaluation.md#part-j--the-feedback-loop-turning-user-signals-into-eval-cases)

## Hands-on
Run a reference-free faithfulness judge on sampled prod traffic; capture thumbs-down → trace → promote to an eval case.

The chapter's full **Hands-on tasks** and **Deliverable** are in [Chapter 7 — Evaluation, Safety & Reliability: shipping with confidence, not vibes](../chapters/07-evaluation.md).

## End-of-day self-test
Test yourself before tomorrow. These come from a bank of real Gen-AI interview questions; the answers are in today's chapter — *peek only after attempting.*

1. What is ROUGE?
2. What is AI evaluation?
3. Did you use BLEU score?
4. Why do LLMs hallucinate?
5. What is hallucination in AI?
6. How does ROUGE accuracy work?
7. Compare ROUGE and BLEU metrics.
8. Explain ROUGE and BLEU metrics.
9. How do you achieve high precision?
10. How to find our model is hallucinating?
11. How do you reduce hallucination in LLMs?
12. How do you prevent hallucination in LLMs?
13. How can you prevent hallucination in LLMs?
14. Explain hallucination in the context of AI.
15. How to handle hallucination in an LLM model?

_(48 topic-matched questions in the bank; 15 shown. See [`ai-eng-questions.md`](../ai-eng-questions.md) for the rest.)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
