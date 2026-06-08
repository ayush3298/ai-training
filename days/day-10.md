# Day 10 — RAG, part 4 — managed pipelines, vector DBs in depth & isolation

> [← Day 9](day-09.md) · [All days](README.md) · [Day 11 →](day-11.md)

**Today's chapter:** [Chapter 4 — Grounding LLMs: RAG & Context](../chapters/04-rag.md)  
**Sections:** Parts J–L  
**Time:** ~2.5–3 hrs

## Goal
Compare a managed RAG pipeline to your hand-built one, go deep on vector databases (ANN indexes, metrics, ops), and enforce multi-tenant isolation.

## Why today matters
At scale, the vector DB and tenant isolation become the system. A retrieval leak across tenants is a security incident, not a bug.

## Study
Work through these sections:

- [Part J — Managed RAG pipelines (cloud-native)](../chapters/04-rag.md#part-j--managed-rag-pipelines-cloud-native)
- [Part K — Vector databases in depth (ANN indexes, metrics, ops & scale)](../chapters/04-rag.md#part-k--vector-databases-in-depth-ann-indexes-metrics-ops--scale)
- [Part L — Multi-tenant retrieval & isolation](../chapters/04-rag.md#part-l--multi-tenant-retrieval--isolation)

## Hands-on
Stand up pgvector in Docker with an HNSW index; compare recall+latency to brute force; add a tenant pre-filter and prove A can't read B's docs.

The chapter's full **Hands-on tasks** and **Deliverable** are in [Chapter 4 — Grounding LLMs: RAG & Context](../chapters/04-rag.md).

## End-of-day self-test
Test yourself before tomorrow. These come from a bank of real Gen-AI interview questions; the answers are in today's chapter — *peek only after attempting.*

1. What is HNSW?
2. What is Qdrant?
3. How does FAISS work?
4. What is a RAG pipeline?
5. Why did you use Pinecone?
6. What are vector databases?
7. Was it recursive chunking?
8. Why would you use Pinecone?
9. Compare Pinecone and FAISS.
10. How do embedding models work?
11. How do you create embeddings?
12. What does vector indexing do?
13. What is vector search retrieval?
14. Why use ANN indexing in Pinecone?
15. How do rerankers work in Pinecone?

_(165 topic-matched questions in the bank; 15 shown. See [`ai-eng-questions.md`](../ai-eng-questions.md) for the rest.)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
