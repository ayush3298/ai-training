# Day 8 — RAG, part 2 — the generation half & when not to

> [← Day 7](day-07.md) · [All days](README.md) · [Day 9 →](day-09.md)

**Module:** RAG & Context  ·  **Time:** ~2.5–3 hrs

## Where we are

_Continues **RAG & Context**. Earlier days covered Parts A, B, C; today picks up where they left off._

---

## Part D — The generation half (turning chunks into a grounded answer)

**11. The grounded-answer prompt: give the model the chunks, and pin it to them.**
Now you assemble the prompt: a system instruction that says *answer only from the provided
context*, the retrieved chunks (clearly delimited — [Chapter 3](03-prompt-engineering.md)), and the question. This is where
[Chapter 3](03-prompt-engineering.md) pays off: delimiters, abstain-licensing, and format control are exactly the tools you
need.
```python
SYSTEM = (
    "You answer questions using ONLY the provided context. "
    "If the context does not contain the answer, say 'I don't have that information.' "
    "Cite the source title in brackets after each claim. Do not use outside knowledge."
)

def answer(question):
    chunks = retrieve(question, k=4)
    context = "\n\n".join(
        f"[{c['meta']['source']}]\n{c['text']}" for c in chunks
    )
    user_msg = f"<context>\n{context}\n</context>\n\nQuestion: {question}"
    resp = client.messages.create(
        model="claude-sonnet-4-6", max_tokens=512, system=SYSTEM,
        messages=[{"role": "user", "content": user_msg}],
        temperature=0,   # grounded Q&A wants determinism, not creativity
    )
    return resp.content[0].text
```
- *Build consequence:* Notice `temperature=0`. RAG answers should be faithful to the source, not
  creative. Low temperature + a strict "only from context" system prompt is the standard
  grounded-Q&A configuration.

**12. Citations and abstention are the features that make RAG *trustworthy*, not just functional.**
Two things separate a toy from a product:
- **Citations:** because you kept `metadata` per chunk, the model can point to the source document
  for each claim — and the user (or you) can verify it. This is the answer to "unverifiable" from
  [Part A](#part-a--the-problem-rag-solves).
- **Abstention:** the *most important* instruction in the system prompt is the license to say "I
  don't know." Without it, when retrieval returns junk, the model falls back to its parametric
  memory and hallucinates a confident wrong answer — the worst failure mode, because it looks
  identical to a right one. With it, a retrieval miss becomes an honest "I don't have that" instead
  of a fabrication.
- *Build consequence:* Always build citations and abstention in from day one. They're not polish;
  they're what makes the system safe to put in front of users.

**13. The end-to-end flow, named.**
Question → `embed(question)` → vector search → top-k chunks → assemble prompt (chunks + question +
grounding system prompt) → model call → grounded answer with citations. Memorize this loop; every
RAG system you ever build or debug is a variation on it.

---

## Part E — Making retrieval actually good (the hard 80%)

A naive top-k cosine search gets you a demo. These are the techniques that get you to production
quality — each fixes a specific, common failure.

**14. Hybrid search — semantic + keyword, because each misses what the other catches.**
Pure semantic search is great at meaning but bad at exact strings: error codes (`ERR_4042`),
product SKUs, names, acronyms. Pure keyword search (BM25) nails those but misses paraphrases.
**Hybrid search runs both and merges the rankings** — the standard production default.
- *Build consequence:* If users search by exact identifiers (codes, IDs, names), pure vector
  search *will* disappoint them. Reach for hybrid.

**15. Reranking — a second, smarter pass over the top results.**
Embedding search is fast but coarse. A **reranker** (a model that scores `(query, chunk)` pairs
directly) is slower but far more accurate at judging true relevance. The pattern: retrieve top-50
cheaply with vectors, then rerank to the best 5. You get vector-search speed *and* high precision.
- *Build consequence:* When "the right doc is in the top 20 but not the top 4," a reranker is the
  fix — cheaper and faster than re-engineering your whole pipeline.
> The *mechanism* behind why a reranker beats embedding search (bi-encoder vs cross-encoder), which
> rerankers to use in 2026, and how to pick an embedding model in the first place are deepened in
> **[Part H](#part-h--choosing-an-embedding-model--cross-encoder-reranking) (concepts 27–30)**.

**16. Query rewriting — fix the question before you search it.**
Users ask short, vague, context-dependent questions (*"what about the second one?"*). Embedding
those directly retrieves poorly. Have a cheap model first rewrite the query into a standalone,
search-optimized form (expand acronyms, resolve "it"/"that" from chat history, split a multi-part
question).
- *Build consequence:* In a *chat* RAG system this is nearly mandatory — follow-up questions are
  meaningless without the conversation's context folded in.

**17. You must *evaluate* retrieval, or you're flying blind.**
"It seemed to work on the three questions I tried" is not evaluation. Build a small set of
`(question, expected-source)` pairs and measure:
- **Retrieval metrics:** does the correct chunk appear in the top-k? (recall@k, hit rate).
- **Answer metrics:** is the final answer correct and faithful to the sources? (this is where
  LLM-as-judge comes in — full treatment in [Chapter 7](07-evaluation.md)).
- *Build consequence:* Every change — new chunk size, new embedding model, adding a reranker —
  gets measured against the eval set. Without it you're guessing, and "improvements" silently make
  things worse. This is the bridge into [Chapter 7](07-evaluation.md).

---

## Part F — When *not* to use RAG (choosing the right tool)

**18. RAG is for knowledge lookup — not for everything.** Know its boundaries:
- **RAG fits** when the answer lives in a large, changing corpus you can search: docs Q&A,
  support, internal knowledge, "chat with this PDF."
- **Long-context instead of RAG:** if the whole relevant source is small enough to fit in the
  window (a single contract, one codebase file), just *paste it in*. RAG is the optimization you
  reach for when it *doesn't* fit. Don't build a vector DB to search one document.
- **Fine-tuning instead of RAG:** RAG teaches the model *facts* ("what's our refund window?").
  Fine-tuning teaches it *behavior/format/style* ("always respond in this house tone/JSON shape").
  They solve different problems and often combine. Full treatment in [Chapter 6](06-customization.md).
- **Agents/tools instead of RAG:** if the answer needs a *live* or *computed* value (today's order
  status, a SQL aggregate, current price), you want a tool call ([Chapter 2](02-apis-and-integration.md) Part D, [Chapter 5](05-agents.md)), not a
  document search.
- *Build consequence:* The senior skill is matching the tool to the need: *static knowledge in a
  corpus* → RAG; *fits in the window* → just paste it; *behavior/style* → fine-tune;
  *live/computed data* → tools. Most real systems blend several.

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. What is GCP RAG?
2. What is Graph RAG?
3. What is agentic RAG?
4. What is vector search?
5. Why is chunking needed?
6. What is fine-tuning in RAG?
7. What is a use case for RAG?
8. In AWS, what are S3 vectors?
9. What are the limitations of RAG?
10. How does vector similarity work?
11. How do you structure a RAG pipeline?
12. Why and when to choose any vector DB?
13. Which embedding models have you used?
14. Have you implemented RAG in a project?
15. Which embedding model is used for RAG?

_(162 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
