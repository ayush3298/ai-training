# Day 17+ — Capstone Project

> **Status: starter skeleton — to be fleshed out.** This file sketches the intended
> shape of the capstone so it can be reviewed and filled in. Sections marked _TODO_
> are placeholders.

## Goal

Each engineer ships **one real LLM-backed feature end-to-end**, exercising the whole
spine (Days 1–16): a prompt or agent, grounded in real data where it helps, wrapped in
an API, evaluated with a real eval set, and deployed behind a backend with logging,
cost tracking, and at least one guardrail.

## Why it matters

The previous chapters each produced an isolated artifact. The capstone proves the team
can **integrate** them into something a user could actually touch — the real test of
"building on top of LLMs."

## Scope guardrails

- Build **on top of** a hosted model via API. Do **not** train or fine-tune a base model.
- Small but complete beats large but half-wired. One feature, fully done.
- Reuse your Day 7–8 RAG, Day 9–10 agent, and Day 13–14 eval harness where they fit.

## Suggested project shapes _(pick one or propose your own)_

- _TODO:_ "chat with our docs" assistant (RAG + citations + abstention)
- _TODO:_ a support-ticket triage agent (classification + tool calls + structured output)
- _TODO:_ an internal "research" agent (multi-tool loop + bounded steps + trace)
- _TODO:_ a structured-extraction service (documents → validated JSON, with eval)

## Required components (the rubric) _(TODO: finalize weights)_

| Area | Requirement | Source chapter |
|------|-------------|----------------|
| Prompt/agent | A versioned prompt or a bounded agent loop | Days 5–6 / 9–10 |
| Grounding | RAG **or** justified decision not to use it | Days 7–8 |
| API | Backend endpoint, key server-side, streaming | Days 3–4 / 15–16 |
| Evaluation | ≥20-case eval set + one metric + ship/no-ship call | Days 13–14 |
| Safety | ≥1 input or output guardrail with a caught case | Days 13–14 |
| Ops | Per-request logging: tokens, cost, latency, version | Days 15–16 |

## Milestones _(TODO: map to calendar days)_

1. _TODO:_ Proposal — problem, users, success metric, chosen project shape
2. _TODO:_ Spike — thinnest end-to-end slice working
3. _TODO:_ Eval set + first measured baseline
4. _TODO:_ Hardening — guardrail, retries, cost lever
5. _TODO:_ Demo + writeup

## Deliverable

- Running feature (repo + run instructions)
- Eval set + before/after numbers + a defended ship/no-ship decision
- One-page writeup: architecture, cost at target traffic, failure modes handled

## Daily update (DM to Ayush)

One line per day: what shipped / where stuck.

## Time estimate

_TODO:_ propose a day-block (suggest Day 17–21).
