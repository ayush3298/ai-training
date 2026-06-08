# Day 4 — APIs, part 2 — structured output, tools & production basics

> [← Day 3](day-03.md) · [All days](README.md) · [Day 5 →](day-05.md)

**Today's chapter:** [Chapter 2 — Talking to an LLM: from first call to production-ready](../chapters/02-apis-and-integration.md)  
**Sections:** Parts C–E  
**Time:** ~2.5–3 hrs

## Goal
Get machine-readable output every time, call a tool, and handle the errors real traffic throws.

## Why today matters
Structured output and tool calling are the foundation for everything in RAG and agents. Error handling is what separates a demo from a service.

## Study
Work through these sections:

- [Part C — Structured output (machine-readable, every time)](../chapters/02-apis-and-integration.md#part-c--structured-output-machine-readable-every-time)
- [Part D — Tool / function calling (the foundation for agents)](../chapters/02-apis-and-integration.md#part-d--tool--function-calling-the-foundation-for-agents)
- [Part E — Production basics (demo → survives real traffic)](../chapters/02-apis-and-integration.md#part-e--production-basics-demo--survives-real-traffic)

## Hands-on
Force schema-valid JSON out of the model; do one tool-calling round trip; add retry/timeout around the call.

Full tasks: [Hands-on for this chapter](../chapters/02-apis-and-integration.md#hands-on).

## End-of-day self-test
Test yourself before tomorrow. These come from a bank of real Gen-AI interview questions; the answers are in today's chapter — *peek only after attempting.*

1. How are the APIs created?
2. How does temperature work?
3. Explain API-based approaches.
4. How do I integrate the OpenAI API?
5. How do streaming responses work from LLMs?
6. Explain the temperature parameter in LLMs.
7. How do you get structured output from OpenAI?
8. What is the temperature parameter in OpenAI/LLMs?
9. What are the different types of APIs in Amazon Lex?
10. Can we use Pydantic for structured output from LLMs?
11. How did you handle LLM API integration in Sonara.ai?
12. What are the key principles of a well-designed REST API?
13. What does the temperature parameter control in AI models?
14. How would you design an API to handle ML models and inference?
15. How can we produce structured outputs when dealing with LangChain?

_(28 topic-matched questions in the bank; 15 shown. See [`ai-eng-questions.md`](../ai-eng-questions.md) for the rest.)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
