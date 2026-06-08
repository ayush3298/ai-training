# Day 9 — RAG, part 3 — ingestion, better embeddings & untrusted text

> [← Day 8](day-08.md) · [All days](README.md) · [Day 10 →](day-10.md)

**Today's chapter:** [Chapter 4 — Grounding LLMs: RAG & Context](../chapters/04-rag.md)  
**Sections:** Parts G–I  
**Time:** ~2.5–3 hrs

## Goal
Master the front of the pipeline (messy docs → clean chunks), choose an embedding model + reranker, and treat retrieved text as untrusted.

## Why today matters
Real corpora are messy PDFs, not clean text. And retrieved text can carry an attack — indirect prompt injection is a RAG-specific threat.

## Study
Work through these sections:

- [Part G — Document ingestion & extraction (the front half of the pipeline)](../chapters/04-rag.md#part-g--document-ingestion--extraction-the-front-half-of-the-pipeline)
- [Part H — Choosing an embedding model + cross-encoder reranking](../chapters/04-rag.md#part-h--choosing-an-embedding-model--cross-encoder-reranking)
- [Part I — Retrieved text is untrusted data (indirect prompt injection in RAG)](../chapters/04-rag.md#part-i--retrieved-text-is-untrusted-data-indirect-prompt-injection-in-rag)

## Hands-on
Run a messy PDF through ingestion+OCR; bake off two embedding models + add a reranker; poison a doc and defend with spotlighting.

The chapter's full **Hands-on tasks** and **Deliverable** are in [Chapter 4 — Grounding LLMs: RAG & Context](../chapters/04-rag.md).

## End-of-day self-test
Test yourself before tomorrow. These come from a bank of real Gen-AI interview questions; the answers are in today's chapter — *peek only after attempting.*

1. Why is OCR used?
2. What are rerankers?
3. What is hybrid search?
4. Why do we do re-ranking?
5. Explain one demerit of RAG.
6. What is the RAG architecture?
7. How does a cross-encoder work?
8. How does vector similarity work?
9. Which chunking strategy and why?
10. Explain an end-to-end RAG pipeline.
11. How do you deploy a RAG application?
12. Which data extractor is used for PDFs?
13. Why is RAG preferred over fine-tuning?
14. What are the types of RAG architecture?
15. How do you include RAG in your project?

_(165 topic-matched questions in the bank; 15 shown. See [`ai-eng-questions.md`](../ai-eng-questions.md) for the rest.)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
