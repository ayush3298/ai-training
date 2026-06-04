## Day 7–8 — Grounding LLMs: RAG & Context

**Goal:** Make the model answer from *your* data — internal docs, a knowledge base, a user's
files — instead of only from what it memorized in pretraining. By the end you can build a
retrieval pipeline end to end: turn documents into searchable vectors, find the right chunks for
a question, feed them to the model, and get an answer that **cites its sources and refuses when
the answer isn't in the data**.

**Why this matters:** Day 2 taught the single most important limit of an LLM: it only knows two
things — what's baked into its weights (frozen at training time, no knowledge of your company,
often months stale) and what's in the context window right now. RAG
(**Retrieval-Augmented Generation**) is the discipline of putting *the right information into the
context window at the right moment*. It is the most common LLM feature in production by a wide
margin — "chat with your docs," support bots, internal search, anything that answers from a
corpus the model never saw. It's also the cheapest, fastest, and lowest-risk way to give a model
new knowledge: no training, no fine-tuning, updates the instant you add a document. If you only
learn one "building on top of LLMs" skill beyond prompting, it's this one.

> **Setup assumed:** same as Day 3 — an API key in your environment and the SDK installed. New
> this block: an embeddings model (same provider, e.g. `voyage` / `text-embedding-3-small`) and a
> vector store. We'll start with a 30-line in-memory store you write yourself (so the mechanism is
> never a black box), then name the real databases you'd use in production.

**Suggested split:** Day 7 = Parts A–C (the problem, embeddings, the retrieval pipeline — build
the "retrieve" half). Day 8 = Parts D–F (generation, making retrieval good, when *not* to use
RAG — build the "generate" half and learn to evaluate it).

---

## Part A — The problem RAG solves

**1. The model's knowledge is frozen, private to nobody, and unverifiable.**
Three separate problems, one root cause — the weights are fixed after training:
- **Stale:** it doesn't know anything that happened after its training cutoff.
- **Ignorant of your world:** it never saw your wiki, your tickets, your codebase, this user's
  documents. No amount of prompting reveals knowledge it doesn't have.
- **Unverifiable:** even when it *is* right, it can't point you to *where* it knows that from. And
  when it's wrong, it's wrong with total confidence (Day 2's "hallucination is the default
  behavior").
- *Build consequence:* For any question whose answer lives in *your* data, a bare model call is
  the wrong tool. You don't need a smarter model — you need to *put the answer in front of it*.

**2. The naive fix — "just paste everything into the prompt" — doesn't scale.**
The instinct is right: context memory beats parametric memory, so put the docs in context. The
problem is volume. Your knowledge base is 10,000 documents; the context window holds maybe a few
hundred pages. You can't paste it all, and even if you could:
- **It's expensive** — you pay per input token on every call (Day 3's cost math), so stuffing
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
  cites them. This is mostly prompt engineering — which you already know from Day 5–6.
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

## Part D — The generation half (turning chunks into a grounded answer)

**11. The grounded-answer prompt: give the model the chunks, and pin it to them.**
Now you assemble the prompt: a system instruction that says *answer only from the provided
context*, the retrieved chunks (clearly delimited — Day 5–6), and the question. This is where
Day 5–6 pays off: delimiters, abstain-licensing, and format control are exactly the tools you
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
  Part A.
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
  LLM-as-judge comes in — full treatment in Section 7).
- *Build consequence:* Every change — new chunk size, new embedding model, adding a reranker —
  gets measured against the eval set. Without it you're guessing, and "improvements" silently make
  things worse. This is the bridge into Section 7 (Evaluation).

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
  They solve different problems and often combine. Full treatment in Section 6.
- **Agents/tools instead of RAG:** if the answer needs a *live* or *computed* value (today's order
  status, a SQL aggregate, current price), you want a tool call (Day 3 Part D, Section 5), not a
  document search.
- *Build consequence:* The senior skill is matching the tool to the need: *static knowledge in a
  corpus* → RAG; *fits in the window* → just paste it; *behavior/style* → fine-tune;
  *live/computed data* → tools. Most real systems blend several.

---

**Resources**
- Anthropic — *Contextual Retrieval* post (chunking + the embed/BM25 hybrid that reduced retrieval
  failures); Anthropic embeddings / Voyage docs.
- OpenAI — embeddings guide + the RAG/retrieval cookbook examples.
- `pgvector` README (Postgres vector search — runs in Docker) and one managed vector DB's
  quickstart (Pinecone/Weaviate/Qdrant) for comparison.

**Hands-on tasks**
1. **Embed & compare:** embed 6 sentences (3 about refunds, 3 about passwords). Print the cosine
   matrix. Confirm same-topic pairs score high, cross-topic low. *Feel* the meaning-space.
2. **Build the index:** take a real doc set (your team wiki, or 5–10 markdown files). Chunk on
   paragraphs with overlap, embed, store the triples in a list. Print chunk count and a sample.
3. **`retrieve()`:** implement the brute-force top-k function. Query it with 5 questions and
   eyeball whether the right chunks come back.
4. **`answer()`:** wire retrieval into a grounded prompt with citations + abstention at
   `temperature=0`. Ask a question the docs answer (check the citation) and one they *don't*
   (confirm it abstains instead of inventing).
5. **Chunking experiment:** re-index at 3 chunk sizes (≈100 / ≈300 / whole-doc tokens). Run the
   same 5 queries. Observe how retrieval quality and token cost change. Write one sentence on the
   tradeoff you saw.
6. **Mini-eval:** write 10 `(question, expected-source)` pairs. Measure hit-rate@4 for two chunk
   sizes. Report which won — with the number.
7. *(Stretch)* **Hybrid:** add a keyword (BM25) score, normalize and combine with the cosine
   score, and find one query that hybrid gets right but pure-vector got wrong.

**Questions**

*Check understanding*
1. What two things does an LLM "know," and which one does RAG add to?
2. Why doesn't "paste all the documents into the prompt" work? Give two distinct reasons.
3. What is an embedding, and what's the one property that makes it useful for search?
4. What does cosine similarity measure, and what does a score near 1.0 vs near 0 mean?
5. Name the two phases of a retrieval pipeline and say which one runs on the user's request path.
6. Why must the query and the documents be embedded with the same model?
7. What are the two halves of a RAG system, and which one is the usual culprit when answers are
   bad?

*Apply it*
8. A user searches for the exact error code `ERR_5021` and pure semantic search returns nothing
   useful. What technique fixes this and why?
9. Your chunks are entire 20-page documents and answers are vague. What's the likely cause and the
   fix?
10. The correct document is reliably in your top-20 results but never the top-4 you paste in.
    What's the cheapest fix?
11. Why is `temperature=0` the usual choice for grounded Q&A?
12. In a multi-turn chat RAG bot, a user asks "what about the refund window for that one?"
    Retrieval returns garbage. What pre-search step is missing?

*Stretch*
13. Without abstention licensing, describe the exact failure that happens when retrieval misses —
    and why it's more dangerous than an obvious error.
14. You're told "RAG is giving wrong answers." Outline the order you'd debug in, and justify why
    retrieval comes before generation.
15. For each, pick RAG / paste-into-context / fine-tune / tool-call and justify: (a) answer from a
    50k-doc knowledge base; (b) summarize one 8-page contract; (c) always reply in your company's
    house JSON format; (d) report a customer's current order status.

**Answer key**
1. Parametric memory (frozen in weights) + context memory (the window). RAG adds knowledge via
   *context* — it puts retrieved facts into the window; it never changes the weights.
2. (a) The corpus is far larger than the context window. (b) Even when it fits, it's expensive
   (per-token cost on every call) and accuracy drops on huge contexts ("lost in the middle").
3. A fixed-length vector of numbers representing text's meaning; the useful property is that
   similar-meaning text maps to nearby vectors regardless of shared words.
4. The angle (direction) between two vectors, ignoring magnitude. ~1.0 = nearly same meaning;
   ~0 = unrelated; negative = opposing.
5. Indexing (offline: chunk→embed→store) and querying (online: embed question→search→top-k).
   Querying is on the request path; indexing is done ahead of time.
6. Different models produce vectors in different, incompatible spaces; cross-model similarity
   scores are meaningless, so retrieval would be garbage.
7. Retrieval (the search) and generation (the model call). Retrieval is the usual culprit — wrong
   chunks in means no prompt can produce a right answer.
8. Hybrid search (semantic + keyword/BM25). Exact identifiers are strings the embedding blurs;
   keyword search matches them exactly while semantic search handles paraphrases.
9. Chunks too large → the answer is diluted by surrounding irrelevant text, weakening the
   similarity score and wasting tokens. Fix: smaller, boundary-aligned chunks with overlap.
10. Add a reranker: retrieve top-N cheaply with vectors, then rerank to the best few. Cheaper than
    re-architecting retrieval.
11. Grounded answers should be faithful to the sources, not creative; temperature 0 minimizes
    invention and makes outputs reproducible.
12. Query rewriting — rewrite the follow-up into a standalone query using chat history (resolve
    "that one," expand the question) *before* embedding/searching.
13. The model silently falls back to parametric memory and produces a confident, fluent, wrong
    answer that's indistinguishable from a correct one — far more dangerous than an explicit error
    because nothing signals the failure.
14. Inspect what `retrieve()` returned first; if the right chunks aren't there, fix
    chunking/search/reranking; only if the chunks *are* correct but the answer is still wrong do
    you touch the generation prompt. Retrieval first because generation can't fix inputs it never
    received.
15. (a) RAG — large searchable corpus. (b) Paste-into-context — it fits in the window; no infra
    needed. (c) Fine-tune — that's behavior/format, not facts. (d) Tool-call — a live, computed
    value, not a static document.

**Deliverable:** a working `chat-with-your-docs` script: index a real doc set, `retrieve()` +
`answer()` with citations and abstention, plus a 10-pair mini-eval reporting hit-rate@4 for two
chunk sizes. It must correctly abstain on an out-of-corpus question.

**Daily update (DM to Ayush):** one line — what you built/learned and any blocker (e.g. "RAG over
the team wiki working; abstention holds on out-of-scope Qs; hit-rate@4 = 0.8 at 300-token
chunks").

**Time:** ~2 days. Day 7: Parts A–C (build the retrieve half). Day 8: Parts D–F (generation,
quality techniques, evaluation, tool-selection judgment).

---

