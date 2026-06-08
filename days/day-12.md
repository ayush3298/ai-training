# Day 12 — Agents, part 2 — state, planning, production & safety

> [← Day 11](day-11.md) · [All days](README.md) · [Day 13 →](day-13.md)

**Today's chapter:** [Chapter 5 — Agents & Tool Use: from single call to autonomous loop](../chapters/05-agents.md)  
**Sections:** Parts D–H  
**Time:** ~3 hrs

## Goal
Add memory/state, planning patterns, and face production reality: cost, latency, failure, managed runtimes, and the tool-boundary safety problem.

## Why today matters
The loop is the easy part. State, cost control, and locking down the tool boundary are what make an agent shippable.

## Study
Work through these sections:

- [Part D — Memory, context & state across steps](../chapters/05-agents.md#part-d--memory-context--state-across-steps)
- [Part E — Planning & multi-step reasoning patterns](../chapters/05-agents.md#part-e--planning--multi-step-reasoning-patterns)
- [Part F — Production reality: cost, latency, safety, failure](../chapters/05-agents.md#part-f--production-reality-cost-latency-safety-failure)
- [Part G — Managed agent runtimes (cloud-native agents)](../chapters/05-agents.md#part-g--managed-agent-runtimes-cloud-native-agents)
- [Part H — Tool & output safety (the two sides of the tool boundary)](../chapters/05-agents.md#part-h--tool--output-safety-the-two-sides-of-the-tool-boundary)

## Hands-on
Add a validation/allowlist gate around tool args; wrap tool output as untrusted; force an injection-via-tool-output and prove the gate holds.

The chapter's full **Hands-on tasks** and **Deliverable** are in [Chapter 5 — Agents & Tool Use: from single call to autonomous loop](../chapters/05-agents.md).

## End-of-day self-test
Test yourself before tomorrow. These come from a bank of real Gen-AI interview questions; the answers are in today's chapter — *peek only after attempting.*

1. What are multi-agents?
2. What is an agent in AI?
3. What is function calling?
4. Explain an agentic AI project.
5. What are agentic AI frameworks?
6. How to evaluate agent accuracy?
7. What is routing in agent workflow?
8. What is function calling in an agent?
9. How to design an agent in Copilot Studio?
10. How would you design a multi-agent system?
11. Does the agent call the tool or vice versa?
12. How do you route agent requests using FastAPI?
13. What kinds of agents are available in Autogen?
14. What framework is used for multi-agent systems?
15. What guardrails can we have in a LinkedIn agent?

_(89 topic-matched questions in the bank; 15 shown. See [`ai-eng-questions.md`](../ai-eng-questions.md) for the rest.)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
