# Day 7 — RAG, part 1 — the retrieve half

> [← Day 6](day-06.md) · [All days](README.md) · [Day 8 →](day-08.md)

**Today's chapter:** [Chapter 4 — Grounding LLMs: RAG & Context](../chapters/04-rag.md)  
**Sections:** Parts A–C  
**Time:** ~2.5–3 hrs

## Goal
Build the retrieval half of a RAG pipeline: the problem it solves, embeddings, and chunk → index → search.

## Why today matters
RAG is how you make the model answer from your data instead of its memory. The retrieve half is where answer quality is won or lost.

## Study
Work through these sections:

- [Part A — The problem RAG solves](../chapters/04-rag.md#part-a--the-problem-rag-solves)
- [Part B — Embeddings & semantic search (the core mechanism)](../chapters/04-rag.md#part-b--embeddings--semantic-search-the-core-mechanism)
- [Part C — The retrieval pipeline (chunking → index → search)](../chapters/04-rag.md#part-c--the-retrieval-pipeline-chunking--index--search)

## Hands-on
Build a brute-force vector store: chunk a corpus, embed it, and retrieve the top-k for a query.

The chapter's full **Hands-on tasks** and **Deliverable** are in [Chapter 4 — Grounding LLMs: RAG & Context](../chapters/04-rag.md).

## End-of-day self-test
Test yourself before tomorrow. These come from a bank of real Gen-AI interview questions; the answers are in today's chapter — *peek only after attempting.*

1. What is GCP RAG?
2. What is agentic RAG?
3. What is vector search?
4. What is fine-tuning in RAG?
5. What is a use case for RAG?
6. What are embeddings in RAG?
7. What is overlapping chunking?
8. How to implement a RAG system?
9. What databases are used for RAG?
10. Why is embedding helpful in RAG?
11. What is RAG and what is its use?
12. When do we use TF-IDF embeddings?
13. How do you evaluate a RAG system?
14. Which embedding model did you use?
15. What other types of RAG are there?

_(165 topic-matched questions in the bank; 15 shown. See [`ai-eng-questions.md`](../ai-eng-questions.md) for the rest.)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
