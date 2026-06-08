# Day 7 — RAG, part 1 — the retrieve half

> [← Day 6](day-06.md) · [All days](README.md) · [Day 8 →](day-08.md)

**Module:** RAG & Context  ·  **Time:** ~2.5–3 hrs

## About this module

### Chapter 4 — Grounding LLMs: RAG & Context

**Goal:** Make the model answer from *your* data — internal docs, a knowledge base, a user's
files — instead of only from what it memorized in pretraining. By the end you can build a
retrieval pipeline end to end: turn documents into searchable vectors, find the right chunks for
a question, feed them to the model, and get an answer that **cites its sources and refuses when
the answer isn't in the data**.

**Why this matters:** [Chapter 1](01-foundations.md) taught the single most important limit of an LLM: it only knows two
things — what's baked into its weights (frozen at training time, no knowledge of your company,
often months stale) and what's in the context window right now. RAG
(**Retrieval-Augmented Generation**) is the discipline of putting *the right information into the
context window at the right moment*. It is the most common LLM feature in production by a wide
margin — "chat with your docs," support bots, internal search, anything that answers from a
corpus the model never saw. It's also the cheapest, fastest, and lowest-risk way to give a model
new knowledge: no training, no fine-tuning, updates the instant you add a document. If you only
learn one "building on top of LLMs" skill beyond prompting, it's this one.

> **Setup assumed:** same as [Chapter 2](02-apis-and-integration.md) — an API key in your environment and the SDK installed. New
> this chapter: an embeddings model (same provider, e.g. `voyage` / `text-embedding-3-small`) and a
> vector store. We'll start with a 30-line in-memory store you write yourself (so the mechanism is
> never a black box), then name the real databases you'd use in production.

---

## Part A — The problem RAG solves

**1. The model's knowledge is frozen, private to nobody, and unverifiable.**
Three separate problems, one root cause — the weights are fixed after training:
- **Stale:** it doesn't know anything that happened after its training cutoff.
- **Ignorant of your world:** it never saw your wiki, your tickets, your codebase, this user's
  documents. No amount of prompting reveals knowledge it doesn't have.
- **Unverifiable:** even when it *is* right, it can't point you to *where* it knows that from. And
  when it's wrong, it's wrong with total confidence ([Chapter 1](01-foundations.md)'s "hallucination is the default
  behavior").
- *Build consequence:* For any question whose answer lives in *your* data, a bare model call is
  the wrong tool. You don't need a smarter model — you need to *put the answer in front of it*.

**2. The naive fix — "just paste everything into the prompt" — doesn't scale.**
The instinct is right: context memory beats parametric memory, so put the docs in context. The
problem is volume. Your knowledge base is 10,000 documents; the context window holds maybe a few
hundred pages. You can't paste it all, and even if you could:
- **It's expensive** — you pay per input token on every call ([Chapter 2](02-apis-and-integration.md)'s cost math), so stuffing
  200k tokens into every request is ruinous.
- **It's slower** and accuracy actually *drops* — models attend worse to information buried in a
  huge context ("lost in the middle"). More context is not more better.
- *Build consequence:* The real task isn't "give the model everything." It's **"find the handful
  of paragraphs that actually answer this question, and give it only those."** That selection step
  *is* retrieval. RAG = a search engine bolted to the front of the model.

**3. RAG in one sentence, and its two halves.**
> **When a question comes in, search your data for the most relevant snippets, paste those
> snippets into the prompt, and ask the model to answer using them.**

Two halves, and you build them separately:
- **Retrieval** (the search engine): given a question, return the top-k most relevant chunks of
  your corpus. This is the hard, unglamorous 80% of the work.
- **Generation** (the model call): given the question + those chunks, write a grounded answer that
  cites them. This is mostly prompt engineering — which you already know from [Chapter 3](03-prompt-engineering.md).
- *Build consequence:* When a RAG system gives a bad answer, **the bug is almost always in
  retrieval, not generation.** If you handed the model the wrong paragraphs, no prompt can save
  it. Debug retrieval first, always.

---

## Part B — Embeddings & semantic search (the core mechanism)

**4. Keyword search isn't enough — you need *meaning* search.**
A user asks *"how do I get my money back?"* The doc says *"refunds are processed within 5 business
days."* Zero shared keywords — classic keyword search (matching words) finds nothing. You need
search that understands *"get my money back"* and *"refund"* mean the same thing. That's
**semantic search**, and it runs on **embeddings**.

**5. An embedding is a piece of text turned into a list of numbers that captures its meaning.**
You send text to an embeddings model; it returns a fixed-length vector (e.g. 1,536 numbers) — a
coordinate in a high-dimensional "meaning space." The one property that makes it useful:
**text with similar meaning lands at nearby coordinates**, regardless of shared words.
- *"how do I get my money back"* and *"refund policy"* → vectors close together.
- *"refund policy"* and *"how do I reset my password"* → vectors far apart.
- *Build consequence:* Embeddings convert the fuzzy human notion of "relevant" into a number you
  can compute. That's the whole trick that makes retrieval possible.

**6. "Closeness" is cosine similarity — a number from -1 to 1.**
To rank chunks by relevance you measure the distance between the question's vector and each
chunk's vector. The standard measure is **cosine similarity**: it looks at the *angle* between two
vectors (direction = meaning), ignoring length. Score near **1.0** = nearly identical meaning;
near **0** = unrelated; negative = opposite. "Find the relevant chunks" becomes, literally, "find
the chunks whose vectors have the highest cosine similarity to the question's vector."
- *Build consequence:* Retrieval ranking is just sorting by a similarity score. You can compute it
  yourself in one line of NumPy — there's no magic in the box.

**Embeddings, two providers:**
```python
# Anthropic ecosystem — Voyage AI (Anthropic's recommended embeddings partner)
import voyageai
vo = voyageai.Client()  # reads VOYAGE_API_KEY
result = vo.embed(["refunds are processed within 5 business days"],
                  model="voyage-3", input_type="document")
vector = result.embeddings[0]      # -> list of floats, e.g. length 1024
```
```python
# OpenAI
from openai import OpenAI
client = OpenAI()
result = client.embeddings.create(model="text-embedding-3-small",
                                  input="refunds are processed within 5 business days")
vector = result.data[0].embedding  # -> list of floats, length 1536
```

**Cosine similarity from scratch (so it's never a black box):**
```python
import numpy as np

def cosine(a, b):
    a, b = np.array(a), np.array(b)
    return a @ b / (np.linalg.norm(a) * np.linalg.norm(b))

# rank chunks against a query
q = embed("how do I get my money back?")
scored = sorted(chunks, key=lambda c: cosine(q, c["vector"]), reverse=True)
top_k = scored[:4]   # the 4 most relevant chunks
```
> **One rule you cannot break:** the query and the documents **must be embedded with the same
> model.** Vectors from different models live in different spaces and their similarity scores are
> meaningless. Pick one embeddings model and use it for both indexing and querying.

---

## Part C — The retrieval pipeline (chunking → index → search)

This is the part you build once and reuse for every RAG project. It has two phases: **indexing**
(offline, done ahead of time) and **querying** (online, per request).

**7. Indexing: chunk → embed → store. Do it once, ahead of time.**
You don't embed whole documents — you split them into **chunks** first (next concept on why),
embed each chunk, and store the `(chunk_text, vector, metadata)` triples in a **vector store**.
This is a batch job you run when documents are added or changed, *not* on the user's request path.
> This concept assumes you already have clean text to chunk. Real corpora arrive as PDFs, Word
> files, scans, and HTML — turning those into clean, structured text is a stage *before* chunking,
> covered in **[Part G](#part-g--document-ingestion--extraction-the-front-half-of-the-pipeline) (Document ingestion & extraction, concepts 19–26)**. If you're building from
> raw files, read [Part G](#part-g--document-ingestion--extraction-the-front-half-of-the-pipeline) first.
- *Build consequence:* Indexing latency is free — it happens offline. Query latency is what the
  user feels. Keep heavy work (embedding the whole corpus) on the indexing side.

**8. Chunking is the highest-leverage decision in the whole pipeline.**
A **chunk** does double duty, and that's *why* it dominates everything downstream:
1. It's the **unit you search over** — you retrieve whole chunks, never half of one. The chunk's
   vector *is* what gets matched.
2. It's the **unit you paste into the prompt** — the model sees exactly the chunk text, nothing
   more, nothing less.

So a bad chunk poisons both halves at once: a chunk that's badly cut retrieves poorly *and*, if it
does get retrieved, hands the model incomplete or noisy context. Every retrieval and every answer
in your system is assembled out of chunks — get the chunks wrong and no embedding model, reranker,
or prompt can recover.

**The core tension: the retrieval ideal and the context ideal pull in opposite directions.**
- For **retrieval precision**, you want chunks **small and single-topic** — one chunk = one idea,
  so its vector points cleanly in one direction and matches a query strongly.
- For **answer completeness**, you want chunks **large enough to be self-contained** — all the
  context needed to understand the point, so the model isn't handed a fragment.

Chunking strategy is the art of satisfying both at once. The two failure cliffs:
- **Too big** (e.g. a whole 20-page doc as one chunk): the vector is an *average* of many topics,
  so it matches everything weakly and nothing strongly — the one relevant sentence is **diluted**
  by pages of unrelated text. You also burn context tokens and money pasting 19 irrelevant pages
  to deliver one paragraph, and you invite "lost in the middle."
- **Too small** (e.g. one sentence per chunk): the answer **splits across chunks** so no single
  chunk is a strong match, and fragments lose the context that gave them meaning — *"It must be
  returned within 30 days."* *What* must? The chunk no longer says, and neither the embedding nor
  the model can tell.

**The default sweet spot:** roughly **a paragraph, or ~200–500 tokens**, split on natural
boundaries. Big enough to be a self-contained thought, small enough to be about *one* thing. Start
here, then tune against your eval set (concept 17) — the right size is **content-dependent** (terse
FAQ entries chunk differently than dense legal prose).

**Chunking strategies, weakest to strongest:**
- **1. Fixed-size (character/token) chunking — the naive baseline.** Cut every N characters/tokens
  (say 500), optionally with overlap. Trivial and fast, but **blind to structure** — it slices
  mid-sentence, mid-word, mid-table, mid-code-block. Use only when text has no structure at all
  (one unbroken transcript), and even then prefer sentence boundaries.
- **2. Sentence / paragraph chunking — respect the smallest natural unit.** Split on sentence or
  paragraph boundaries so every chunk is at least a complete thought. Limit: paragraph sizes vary
  wildly, so you usually want to *pack* small paragraphs together up to a target size.
- **3. Recursive character chunking — the practical default.** Try the **largest** natural
  separator first and fall back to smaller ones only if a piece is still too big — a typical ladder
  is `["\n\n" → "\n" → ". " → " "]`. It adapts to the document (well-structured docs split on
  paragraphs; dense walls of text degrade gracefully to sentences). This is the go-to first choice.
- **4. Document-structure-aware chunking — use the format's own skeleton.** Structured formats hand
  you boundaries for free: **Markdown** → split on headings, keep the heading with its section;
  **HTML** → `<section>`/`<h*>`; **code** → function/class boundaries (never cut a function in
  half); **PDF** → detected headings, keep page numbers in metadata. The author already grouped
  related content under each heading — honoring that produces coherent chunks by construction, and
  the heading is great context to prepend.
- **5. Semantic chunking — let meaning decide the cut.** Embed sentences one by one and start a new
  chunk wherever the topic shifts (a large drop in consecutive-sentence similarity). Maximally
  single-topic, but you embed at sentence level just to *decide boundaries* — more expensive and
  complex. Reach for it only when content rambles across topics without clear headings.

**Techniques that wrap around whichever strategy you pick:**
- **Overlap — insurance against boundary cuts.** Let consecutive chunks share ~**10–20%** of their
  text (repeat the last sentence or two of the previous chunk). An idea that straddles a boundary
  then survives intact in at least one chunk. Don't overdo it — too much overlap inflates the index
  and returns near-duplicate chunks.
- **Metadata — attach it at index time; you can't reconstruct it later.** Every chunk should carry
  `source`/`title`, `url`, `section`/`heading`, `date`, plus domain fields (`product`, `doc_type`).
  You need it for **citations** (concept 12), **filtering** ("search only *this* product, after
  *this* date" — a metadata `WHERE` combined with the vector search), and **debugging** ("which doc
  did this bad chunk come from?").
- **Contextual chunking (contextual retrieval) — the high-impact upgrade.** An isolated chunk loses
  the context of its document — *"The deductible is $500."* for *which* plan, *which* year? The fix:
  **prepend a short, chunk-specific context blurb before embedding and storing**, e.g. `"From the
  2024 Premium Health Plan policy, Deductibles section: The deductible is $500."` The title +
  heading path is often enough, or have a small model write a one-line situating sentence per chunk.
  Anthropic's *Contextual Retrieval* work showed this meaningfully cuts retrieval failures, because
  every chunk now stands on its own. Pairs especially well with structure-aware chunking.
- **Parent–child / "small-to-big" retrieval — search small, feed big.** Decouple the chunk's two
  jobs: **index small** chunks (precise matching) but, once one is retrieved, **paste its larger
  parent** (the full paragraph/section) into the prompt. You get small-chunk retrieval precision
  *and* large-chunk answer completeness — sidestepping the size tradeoff instead of compromising.
  Store a `parent_id` on each child chunk.

**Special content that breaks naive chunking:**
- **Tables:** never split across chunks — rows get orphaned from their headers and become
  meaningless. Keep a table whole; if huge, repeat the header row or convert to markdown / one
  row-as-a-sentence per record.
- **Code:** split on function/class boundaries; keep a function with its signature and docstring. A
  half-function is useless to retrieval and reader alike.
- **Lists / steps:** keep an ordered procedure together — step 4 alone (*"Then click Confirm."*) is
  unactionable.
- **Repeated boilerplate** (headers, footers, nav, per-page legal text): strip it *before* chunking,
  or every chunk's vector gets muddied by the same noise and they all start matching each other.

**How to choose — a practical decision path:**
1. **Start with recursive chunking** at ~300 tokens, ~15% overlap. Robust default.
2. **If your docs have structure (Markdown/HTML/code), switch to structure-aware** chunking and keep
   headings — usually the single biggest quality jump.
3. **Add contextual prefixes** (title + heading path, or a generated one-liner) before embedding —
   high impact, low effort.
4. **If precision-vs-completeness is fighting you, adopt parent–child** (index small, return big).
5. **Only reach for semantic chunking** if content rambles without clear boundaries and the above
   isn't enough — it's the most expensive option.
6. **Measure every change against your eval set** (hit-rate@k). Chunking is empirical: the "right"
   answer is whichever config scores best on *your* corpus, not a number from a blog post.

```python
# Recursive, structure-respecting chunker with overlap — the practical default.
def split_into_chunks(text, target_tokens=300, overlap=0.15):
    seps = ["\n\n", "\n", ". ", " "]          # coarsest → finest boundary

    def split(t, seps):
        if estimate_tokens(t) <= target_tokens or not seps:
            return [t]
        head, *rest = seps
        parts, out = t.split(head), []
        for p in parts:                        # recurse into any still-too-big piece
            out.extend(split(p, rest) if estimate_tokens(p) > target_tokens else [p])
        return out

    raw = split(text, seps)
    # pack small pieces up toward the target, then stitch overlap between neighbors
    return add_overlap(pack(raw, target_tokens), overlap)
```
- *Build consequence:* When RAG gives bad answers, **fix chunking before rerankers, hybrid search,
  or a bigger model.** Most "it retrieved the wrong thing" bugs are really "the right thing was
  never a clean, self-contained, well-sized chunk." It's the cheapest, highest-leverage knob you
  have — and the upgrade order that pays off fastest is *structure-aware → contextual prefixes →
  parent–child → (only if needed) semantic*.

**9. The vector store: what it is, and what you'd use in production.**
A vector store holds your chunks' vectors and answers one question fast: *"given this query
vector, what are the k nearest chunks?"* At small scale that's a brute-force loop (compute cosine
against every chunk). At real scale (millions of vectors) you use an **ANN** (approximate nearest
neighbor) index that finds the near-neighbors without comparing against all of them.
- **Learn-it-yourself:** a Python list + the NumPy loop above. Fine for hundreds of chunks.
- **Production options:** dedicated vector DBs (Pinecone, Weaviate, Qdrant, Chroma) or `pgvector`
  (a Postgres extension — run it in Docker). They add persistence, metadata filtering, and fast
  ANN search.
- *Build consequence:* Don't start with a vector DB. Build the brute-force version first so you
  understand exactly what the DB is doing for you, then swap it in when scale demands.

**10. Querying: embed the question → search → return top-k. This is the per-request hot path.**
```python
# --- INDEXING (run once, offline) ---
store = []
for doc in documents:
    for chunk in split_into_chunks(doc["text"], target_tokens=300, overlap=0.15):
        store.append({
            "text": chunk,
            "vector": embed(chunk),                 # same embed() used for queries
            "meta": {"source": doc["title"], "url": doc["url"]},
        })

# --- QUERYING (per user request) ---
def retrieve(question, k=4):
    qv = embed(question)
    ranked = sorted(store, key=lambda c: cosine(qv, c["vector"]), reverse=True)
    return ranked[:k]
```
- *Build consequence:* The interface every RAG system exposes is exactly this:
  `retrieve(question) -> list[chunk]`. Everything in Part B–C is the implementation behind that one
  function.

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. How does RAG work?
2. What is the RAG model?
3. What is Corrective RAG?
4. Explain semantic chunking.
5. What is overlapping chunking?
6. Have you utilized agentic RAG?
7. What is an agentic RAG system?
8. Why use RAG instead of memory?
9. How are embeddings used in LLMs?
10. What databases are used for RAG?
11. How do you use Cloud Run for RAG?
12. When do we use TF-IDF embeddings?
13. What are some chunking strategies?
14. Which embedding model did you use?
15. What other types of RAG are there?

_(163 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
