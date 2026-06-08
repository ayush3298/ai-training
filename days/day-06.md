# Day 6 — Prompting, part 2 — debugging, prompts-in-code, injection & PII

> [← Day 5](day-05.md) · [All days](README.md) · [Day 7 →](day-07.md)

**Today's chapter:** [Chapter 3 — Prompt Engineering: reliable behavior on demand](../chapters/03-prompt-engineering.md)  
**Sections:** Parts D–G  
**Time:** ~2.5–3 hrs

## Goal
Treat prompts as versioned code, debug them methodically, and build the injection + PII-redaction reflexes.

## Why today matters
Prompts ship to production. They need versioning, a debugging method, and a security posture like any other code.

## Study
Work through these sections:

- [Part D — Iterating & debugging a prompt](../chapters/03-prompt-engineering.md#part-d--iterating--debugging-a-prompt)
- [Part E — Prompts in code (templating & caching)](../chapters/03-prompt-engineering.md#part-e--prompts-in-code-templating--caching)
- [Part F — Prompt injection & safety](../chapters/03-prompt-engineering.md#part-f--prompt-injection--safety)
- [Part G — Multimodal prompting](../chapters/03-prompt-engineering.md#part-g--multimodal-prompting)

## Hands-on
Put a prompt under version control with caching; redact PII before a call and restore it after; break a prompt with an injection, then defend it.

Full tasks: [Hands-on for this chapter](../chapters/03-prompt-engineering.md#hands-on).

## End-of-day self-test
Test yourself before tomorrow. These come from a bank of real Gen-AI interview questions; the answers are in today's chapter — *peek only after attempting.*

1. What are prompt types?
2. How do you prevent prompt injection?
3. How many types of prompting are there?
4. Write a prompt for key-value pair mapping.
5. What are the different types of prompting?
6. How do you handle prompt injection attacks?
7. How do you protect against prompt injection?
8. What are suggested prompts in Copilot Studio?
9. What is the prompting strategy in this project?
10. How do you design a prompt for structured output?
11. How to use Gen AI prompts in version 10 of Kore.ai?
12. What are the types of prompting and their benefits?
13. How do you prevent prompt-driven escalation in LLMs?
14. How do you create prefabricated prompts in Copilot Studio?
15. What is chain-of-thought prompting and how do you apply it?

_(40 topic-matched questions in the bank; 15 shown. See [`ai-eng-questions.md`](../ai-eng-questions.md) for the rest.)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
