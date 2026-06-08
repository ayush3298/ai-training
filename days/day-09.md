# Day 9 — RAG, part 3 — ingestion, better embeddings & untrusted text

> [← Day 8](day-08.md) · [All days](README.md) · [Day 10 →](day-10.md)

**Module:** RAG & Context  ·  **Time:** ~2.5–3 hrs

## Where we are

_Continues **RAG & Context**. Earlier days covered Parts A, B, C, D, E, F; today picks up where they left off._

---

## Part G — Document ingestion & extraction (the front half of the pipeline)

Parts B–C started from a clean string of text. Real corpora don't. They arrive as PDFs (born-digital
*and* scanned), Word docs, HTML, EPUB, and images. **Ingestion** is the stage *before* chunking:
turning real files into clean, structured, chunkable text. It's the most-skipped and most
under-budgeted stage in RAG, and it silently caps the quality of everything downstream.

**19. You embed *text*, never the file.** The vector store holds vectors of extracted text — it
never holds the PDF. So the first question for any source is "what text comes out of this, and is it
all of it?" The trap is **image-only content**: a scanned page, a screenshot, a diagram, a photo of
a form. To a text extractor that page is *blank* — it contributes nothing to retrieval until you
**OCR it** (pixels → characters), **vision-extract it** (a vision-LLM reads the image and describes
or transcribes it), or **index it as an image** (multimodal embeddings — see the cross-reference at
the end of this Part). "Is the PDF itself stored in the vector DB?" — no, you extract text first.
"Page 3 is all images" — page 3 contributes nothing to retrieval unless you process it.
- *Build consequence:* Before anything else, audit your sources for image-only content. Any page
  your extractor returns empty is a silent hole in your knowledge base — the model can't retrieve
  what was never extracted.

**20. OCR is not layout understanding — RAG needs both.** **OCR** answers "what characters are on
this page?" **Layout understanding** answers "what's the *reading order*, where are the columns,
which lines are headings, where do tables and figures start and end?" A two-column page OCR'd
without layout awareness interleaves the two columns into nonsense; a table OCR'd flat loses which
cell belongs to which header. You need the characters *and* the structure, because chunking ([Part C](#part-c--the-retrieval-pipeline-chunking--index--search))
runs on structure — headings, paragraphs, table boundaries.
- *Build consequence:* When evaluating an extraction tool, test it on a *multi-column, table-heavy*
  page, not a clean single-column one. Plain OCR accuracy on easy pages tells you nothing about
  whether structure survives.

**21. The four tool families, and the five axes you choose on.** There's no single best extractor —
you pick per corpus along five axes: **born-digital vs scanned**, **layout/table complexity**,
**volume/cost**, **determinism/compliance** (can you reproduce the same output and audit it?), and
**privacy** (can the data leave your network?). The landscape:
- **(1) Pure text extractors** — Apache Tika, PyMuPDF. Fast, free, deterministic; pull the embedded
  text layer straight out of born-digital files. They return **blank on scans** (no text layer to
  pull) and don't reconstruct complex layout.
- **(2) Open-source layout + OCR parsers** — Docling, Marker, Unstructured. Self-hostable (privacy +
  no per-page fee), do layout detection + OCR, and emit **Markdown or typed elements** (heading /
  paragraph / table / list) that feed structure-aware chunking directly.
- **(3) Managed cloud IDP APIs** — AWS Textract (best-in-class **tables**; limitation: you can't
  supply your own training data), Azure AI Document Intelligence (formerly Form Recognizer; best when
  you need to **train a custom extractor** for your own form types), Google Document AI (strongest on
  **low-quality scans**). Pay per page, data leaves your network, very high accuracy.
- **(4) Enterprise IDP** — ABBYY and similar. Heavyweight, on-prem options, deep document-processing
  feature sets for regulated/high-volume shops.
- **(5) Vision-LLM extraction** — feed each **page as an image** to GPT-4o / Claude / Gemini and ask
  for JSON. Wins on **messy, varied, never-seen-before layouts** and anything needing *reasoning*
  about the page. But it's **probabilistic, slow, and expensive at scale** (rough order: ~$50k vs
  ~$750 to process 500k pages versus deterministic OCR) and it can **scramble structure** — e.g.
  mixing the sections of one resume together. Use it surgically, not as the default for every page.
- *Build consequence:* Match the family to the corpus before writing code: clean born-digital docs →
  pure extractor; self-host/privacy → open-source parser; perfect tables or custom forms or bad
  scans → the matching cloud IDP; messy one-off layouts → vision-LLM. The wrong family wastes either
  accuracy or money by an order of magnitude.

**22. Tables are the hardest element — handle them atomically.** This is the concrete *how* behind
concept 8's rule "never split a table." Detect the table as one unit, **extract it atomically**,
**serialize it to Markdown or HTML** (so the structure survives into the chunk and the model can
read it), and on a table that spans pages, **repeat the header row** on each piece so no continuation
fragment is orphaned from its column names. Tables are exactly where cheap extractors fail quietly:
the numbers come out, but bound to the wrong headers.
- *Build consequence:* If your corpus is table-heavy (finance, specs, pricing, inventory), table
  fidelity is the *first* thing to evaluate, and often the single reason to escalate from a free
  extractor to Textract or a vision-LLM.

**23. For domain documents, extract to a schema — don't dump to free text.** Invoices, resumes,
inventories, forms have a *known shape*. For these, the right move is often **schema-first
extraction** (structured outputs / function-calling to a target JSON schema), not free-text RAG. You
get typed, queryable fields instead of a paragraph blob. This is what powers, e.g., **resume ↔
job-description matching**: extract each résumé into structured sections, store **per-section
metadata**, and use **small-to-big** (concept 8) to retrieve the *right* section (the "Experience"
block, not the "Education" block) rather than the whole CV.
- *Build consequence:* Ask "does this document type have a known shape?" If yes, extract to a schema
  and you turn fuzzy retrieval into exact field lookup — faster, cheaper, and verifiable.

**24. Extraction must be evaluated like retrieval — on a gold set.** Extraction is not a step you do
once and trust. Build a **gold set** (a handful of documents with the correct output hand-verified)
and measure **field accuracy**, **table-cell accuracy**, and **character accuracy** against it. Skip
this and **errors cascade**: a mangled table or a dropped scanned page silently caps the ceiling of
the whole system, and you'll waste days tuning chunk size and embeddings to fix a problem that lives
two stages upstream.
- *Build consequence:* Stand up an extraction eval *before* a retrieval eval. Garbage in, garbage
  out — and the garbage is invisible unless you measure it at the source.

**25. The 2026 production default is a hybrid, routed pipeline.** You don't pick one tool for the
whole corpus — you **route**. Run cheap, deterministic OCR/extraction on the standard ~90% of pages,
and **escalate the messy ~10%** (failed-confidence pages, complex tables, bad scans, odd layouts) to
a vision-LLM. You get deterministic-OCR economics on the bulk and vision-LLM robustness exactly where
it's needed.
- *Build consequence:* Design ingestion as a router from day one, with a confidence/complexity signal
  deciding which pages escalate. A flat "vision-LLM everything" bill is unaffordable; a flat "cheap
  OCR everything" pipeline quietly drops your hardest, often most valuable, pages.

**26. Ingestion is the front half of the pipeline — get the text right first.**
- *Build consequence:* Before you tune chunk size, embedding model, or reranker, make sure the text
  you're chunking is **correct, complete, and structure-preserving**. No downstream technique can
  recover information that extraction lost.

> **Forward pointer:** everything above extracts text *out* of documents. The alternative — skipping
> extraction and retrieving over **page-as-image** directly (OCR-free multimodal embeddings like
> ColPali), plus specialized formats like DICOM — is covered in the *Beyond Text* extension chapter.

---

## Part H — Choosing an embedding model + cross-encoder reranking

Concepts 5–6 covered what an embedding is, cosine, and the same-model rule; concepts 14–15 introduced
hybrid search and rerankers. This Part deepens those into *decisions*: which embedding model to pick,
and the mechanism that makes the retrieve-then-rerank pipeline universal. (It does **not** re-cover
ANN indexes, distance-metric internals, or quantization — those belong to the separate vector-DB
graft on the roadmap. This Part is about **model choice + reranking**.)

**27. Choosing an embedding model — what actually differs, and how to decide.** Embedding models
differ on: **dimension** (vector length → storage + search cost), **max input length** (how big a
chunk it'll embed), **multilingual** coverage, **cost per 1M tokens**, **MTEB score** (a public
benchmark), and **self-host vs API**. Read the **MTEB leaderboard as a *starting signal*, never a
verdict** — a model's rank on generic benchmarks rarely matches its rank on *your* corpus, so you
**validate the top two or three on your own eval set** (concept 17). The 2026 field:
- **OpenAI `text-embedding-3`** — the safe default; `-small` is the cheap workhorse, `-large` is
  near-top quality.
- **Voyage, Cohere, Gemini** — top-scoring API embeddings (Voyage is Anthropic's recommended partner).
- **BGE-M3, Qwen3-Embedding, E5, NV-Embed** — strong **open-weight** models you self-host for privacy
  or to kill per-token cost.
- *Build consequence:* Pick two or three candidates off the leaderboard within your constraints
  (budget, privacy, language), then let *your* hit-rate@k pick the winner. The benchmark narrows the
  field; your data makes the call.

**28. Matryoshka embeddings — dimension is now a tunable knob, not a permanent decision.** Most 2026
embedding models (OpenAI `-3`, Gemini, Voyage, Qwen3) are trained with **Matryoshka Representation
Learning (MRL)**: the model emits one vector whose *most important information is packed into the
leading dimensions*, so you can **truncate** it (1536 → 512 → 256) and keep most of the quality. A
shorter vector means proportionally less storage and faster search, at a small, measurable accuracy
cost.
- *Build consequence:* "What dimension?" stopped being a one-way door. Embed once at full dimension,
  then truncate to the shortest length that still passes your hit-rate@k — buying large storage and
  latency savings for a tradeoff you can *measure* instead of guess.

**29. Bi-encoder vs cross-encoder — the mechanism behind retrieve-then-rerank.** This is *why*
concept 15's reranker works:
- An **embedding model is a bi-encoder**: query and document are encoded **independently** into
  vectors, then compared with cosine. Because each document's vector never depends on the query, you
  **pre-compute** them all at index time and, at query time, just embed the query and do fast
  similarity math. This is what scales to millions of chunks — but the model **never sees the query
  and document together**, so it can't reason about their specific interaction.
- A **cross-encoder reranker** concatenates **`[query, document]` into one sequence** and runs a full
  transformer pass with **cross-attention** between them, emitting a single relevance score. It judges
  the *pair* directly, so it's far more accurate — but it **can't be pre-computed** (the score depends
  on the query) and costs **O(candidates)** model passes per query.
- Hence the **universal pipeline**: the bi-encoder fetches the top ~50 cheaply, the cross-encoder
  reranks those down to the top ~5 precisely. You spend the expensive cross-attention only on a tiny
  candidate set. 2026 rerankers: **Cohere Rerank, Voyage Rerank** (API); **BGE-reranker, FlashRank**
  (self-host).
- *Build consequence:* Reserve the cross-encoder for a short candidate list, never the whole corpus.
  Bi-encoder for recall at scale, cross-encoder for precision on the shortlist — that division of
  labor *is* the production retrieval pipeline.

**30. Lexical vs embedding retrieval — a decision table.** This makes concept 14's hybrid paragraph
crisp. (Note: **fuzzy/string matching** — edit-distance, surface-form similarity like "color" vs
"colour" — matches *characters*; **embedding similarity** matches *meaning*. They are not the same
axis.)

| Axis | Lexical (BM25 / keyword) | Embedding (semantic) |
|------|--------------------------|----------------------|
| What it matches | exact words / tokens | meaning |
| Typos / misspellings | brittle (misses) | fairly robust |
| Exact codes / SKUs / IDs | **excellent** | weak (blurs them) |
| Paraphrases / synonyms | weak (no shared words) | **excellent** |
| Cost / infra | cheap, no model | embedding model + vector store |
| When it wins | identifiers, names, rare terms, exact quotes | conceptual questions, "how do I…" |

- *Build consequence:* These two fail on opposite inputs, which is exactly why **hybrid search**
  (concept 14) runs both and merges — and why a corpus full of identifiers needs the lexical leg, not
  just bigger embeddings.

---

## Part I — Retrieved text is untrusted data (indirect prompt injection in RAG)

**31. Indirect prompt injection — the live RAG threat.** **Direct** injection (the *user* types
"ignore previous instructions…") is well known. The serious 2026 threat in RAG is **indirect
injection**: malicious instructions **hidden inside a document you retrieve** (or a tool output, a
web page, an email, an image). Your pipeline dutifully fetches that chunk and pastes it into the
prompt — and now attacker text sits in the model's context with the same apparent authority as your
own instructions. This is **OWASP LLM01** (prompt injection, including the indirect variant) plus
**LLM08** (vector/embedding weakness — "RAG poisoning," where an attacker plants a document
*designed* to be retrieved). The root problem is structural: **instructions and data share one token
stream**, so there is no clean escape character that says "everything past here is just data."
- *Build consequence:* Treat **every retrieved chunk as untrusted input**, exactly like raw user
  input — never as trusted instructions. Retrieval is an attack surface, not a safe internal channel.

**32. Defending RAG — segregate system from retrieved content.** No single trick fully solves it;
you layer defenses:
- **Authority separation:** retrieved content goes in at **data** authority, never system-prompt
  authority. Maintain a clear **instruction hierarchy — system > user > tool/retrieved** — and
  instruct the model that nothing in the context can override the system prompt.
- **Spotlighting / delimiting / labelling:** wrap retrieved chunks in explicit markers (e.g.
  `<untrusted_context>…</untrusted_context>`) and tell the model that text inside is *reference
  material to answer from, never commands to obey*.
- **An injection classifier as an input rail:** screen retrieved chunks for instruction-like content
  before they reach the model.
- **Combine with multi-tenant isolation** (the other Chapter 4 graft) so retrieval can't surface
  out-of-scope or attacker-planted documents in the first place — the cheapest injection to defend is
  the one you never retrieve.

This is the concrete answer to "how do you segregate the system prompt from the user/retrieved
content in RAG": you can't *physically* separate them in the token stream, so you separate them by
**labeling, hierarchy, and screening**.
- *Build consequence:* Bake spotlighting + an instruction hierarchy into your grounded-answer prompt
  (concept 11) from the start; retrofitting trust boundaries after a leak is far more painful.

> **Cross-reference:** the architectural defenses (dual-LLM / CaMeL patterns) live in the
> *Security, Privacy & Governance* chapter, and tool-output safety in the Chapter-2/Section-5 tool-use
> material. Keep the deep treatment there — this Part just establishes that *retrieved text is
> untrusted*.

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. Why is OCR used?
2. What are rerankers?
3. What is hybrid search?
4. How do you create embeddings?
5. What does vector indexing do?
6. What is the RAG architecture?
7. How does a cross-encoder work?
8. Why is embedding helpful in RAG?
9. Did you use a RAG-based model? How?
10. Explain an end-to-end RAG pipeline.
11. Which embedding model should be used?
12. Which data extractor is used for PDFs?
13. What are the types of RAG architecture?
14. How do you include RAG in your project?
15. What are cross-encoders and bi-encoders?

_(162 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
