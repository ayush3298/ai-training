# AI Training Plan

A top-down program for the team to learn modern AI / LLMs — start from the big picture,
then go deeper section by section. Daily updates go to Ayush via Slack DM (a one-liner is fine).

## Who this is for

We are training engineers to **build on top of LLMs** — to develop products and features that
use models via APIs — **not** to build or train LLMs themselves.

That sets the depth: internals (pretraining, the Transformer, reinforcement learning) are
covered only deep enough to make good engineering decisions. We never train a model. The real
depth of this program is in sections 2–9 (prompting, APIs, RAG, agents, adaptation, evaluation,
deployment). Every Foundation topic is taught through one lens: **"how does this change
something I'd build?"**

## Course spine

| # | Section | Day-block | Chapter |
|---|---------|-----------|---------|
| 1 | Foundations — How LLMs Actually Work | Day 1–2 | [01 — Foundations](chapters/01-day-1-2-foundations.md) |
| 2 | Using LLMs Well — Prompting & Interaction | Day 5–6 | [03 — Prompt Engineering](chapters/03-day-5-6-prompt-engineering.md) |
| 3 | Building with LLMs — APIs & Integration | Day 3–4 | [02 — APIs & Integration](chapters/02-day-3-4-apis-and-integration.md) |
| 4 | Grounding LLMs — RAG & Context | Day 7–8 | [04 — RAG](chapters/04-day-7-8-rag.md) |
| 5 | Agents & Tool Use | Day 9–10 | [05 — Agents](chapters/05-day-9-10-agents.md) |
| 6 | Customization — Fine-tuning & Adaptation | Day 11–12 | [06 — Customization](chapters/06-day-11-12-customization.md) |
| 7 | Evaluation, Safety & Reliability | Day 13–14 | [07 — Evaluation](chapters/07-day-13-14-evaluation.md) |
| 8 | Deployment & Production | Day 15–16 | [08 — Deployment](chapters/08-day-15-16-deployment.md) |
| 9 | Capstone Project | Day 17+ | _to be planned_ |

> The spine is ordered by topic; the day-blocks run in teaching order (APIs come early, on
> Day 3–4, so there's something concrete to prompt against). Read the chapters in day order:
> 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08.

## Chapters

1. [Day 1–2 — Foundations: how LLMs actually work](chapters/01-day-1-2-foundations.md)
2. [Day 3–4 — Talking to an LLM: from first call to production-ready](chapters/02-day-3-4-apis-and-integration.md)
3. [Day 5–6 — Prompt Engineering: reliable behavior on demand](chapters/03-day-5-6-prompt-engineering.md)
4. [Day 7–8 — Grounding LLMs: RAG & Context](chapters/04-day-7-8-rag.md)
5. [Day 9–10 — Agents & Tool Use: from single call to autonomous loop](chapters/05-day-9-10-agents.md)
6. [Day 11–12 — Customization: adapting a model without training one](chapters/06-day-11-12-customization.md)
7. [Day 13–14 — Evaluation, Safety & Reliability: shipping with confidence](chapters/07-day-13-14-evaluation.md)
8. [Day 15–16 — Deployment & Production: running an LLM feature for real](chapters/08-day-15-16-deployment.md)
9. Day 17+ — Capstone Project _(to be planned)_

## How each chapter is built

Every day-block follows the same shape so the team always knows where to look:

- **Goal** → **Why it matters** → a **setup-assumed** note
- **Suggested split** across the two days
- **Parts** (numbered concepts) — each concept ends with a *Build consequence:* line tying it
  back to something you'd actually build
- Side-by-side **Anthropic / OpenAI** Python where code applies
- **Resources** → **Hands-on tasks** → **Questions** (Check understanding / Apply it / Stretch)
  → **Answer key** → **Deliverable** → **Daily update (DM to Ayush)** → **Time estimate**
