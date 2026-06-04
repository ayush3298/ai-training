## Chapter 1 — Foundations: How LLMs Actually Work

### Session 1 — Watch the foundation video

**Goal:** Get the full mental model of how LLMs work, end to end. Exposure, not mastery —
we go back and deepen each section on later days.

**Task:** Watch the whole video, start to finish.
- **Title:** *Deep Dive into LLMs like ChatGPT* — Andrej Karpathy
- **Link:** https://www.youtube.com/watch?v=7xTGNNLPyMI
- **Length:** ~3.5 hours (a full day's commitment; one pass is fine)

**What the video covers** (this is the spine of this chapter (Foundations)):

| # | Sub-topic | Video segment |
|---|-----------|---------------|
| 1.1 | The Big Picture — what's behind the text box, "predict the next token" | 0:00–1:02 |
| 1.2 | Pretraining & Data — crawling the internet, filtering, FineWeb, 15T tokens | 1:02–7:51 |
| 1.3 | Tokenization — bytes → BPE → ~100k vocab; why models see tokens, not letters | 7:51–14:28 (revisited 2:01:13) |
| 1.4 | The Neural Network — inputs/outputs, the Transformer, parameters, attention | 14:28–26:02 |
| 1.5 | Inference — sampling, stochasticity, temperature | 26:02–31:12 |
| 1.6 | The Base Model — GPT-2 vs Llama 3.1, internet simulator, in-context learning | 31:12–59:28 |
| 1.7 | Post-Training → Assistant (SFT) — conversation data, human labelers | 59:28–1:20:33 |
| 1.8 | LLM Psychology — hallucinations, tool use, "models need tokens to think," jagged intelligence | 1:20:33–2:07:25 |
| 1.9 | Reinforcement Learning — reasoning models, DeepSeek-R1, AlphaGo, RLHF | 2:07:25–3:09:38 |
| 1.10 | Mental Model & Practical Use — grand summary, Swiss-cheese capabilities, use as a tool | 3:09:38–end |

**Deliverable:** None to produce. Optional: jot down "things I didn't get" — it makes the
deeper days easier.

**Daily update:** where you got to — e.g. "Watched it" or
"got to ~2hr, finishing tomorrow."

---

### Session 2 — The Builder's Mental Model of an LLM

**Goal:** Turn Session 1's exposure into a precise, working mental model — the handful of ideas
that actually change how you design and debug an app built on an LLM. By the end you should be
able to explain, to a teammate, *why* an LLM behaves the way it does and what that forces you
to do as a builder.

**Why this session matters:** Most bugs and bad design choices in LLM apps come from a wrong mental
model — treating the model like a database, a deterministic function, or a person that "knows"
things. Get these six ideas right and most of the later material (prompting, RAG, evals, cost
control) becomes obvious instead of mysterious.

---

#### Core concepts

**1. An LLM is a next-token predictor over tokens.**
The model takes a sequence of tokens and outputs a probability distribution over the next
token, then samples one, appends it, and repeats. There is no plan, no database lookup, no
"thinking" beyond this loop (except what later sections add via tools/reasoning).
- *Build consequence:* The model doesn't "answer questions" — it continues text in a way that
  is statistically plausible. Your prompt's job is to make the desired answer the most
  plausible continuation. Cost and latency are both functions of token count (in + out).

**2. Tokenization — the model sees tokens, not characters or words.**
Text is chopped into sub-word chunks (e.g. GPT-4 uses ~100k distinct tokens). " hello" and
"Hello" can be different tokens; numbers and rare words split oddly; non-English text and code
often use *more* tokens per idea.
- *Build consequence:* (a) Context limits and pricing are measured in tokens, not words —
  ~0.75 words/token for English is a rough guide. (b) Character-level tasks (spelling, counting
  letters, reversing a string) are unreliable because the model can't "see" letters. (c) A
  Hindi or code-heavy prompt can cost noticeably more than the same content in English.

**3. The context window IS the model's working memory.** *(The most important idea today.)*
The model knows exactly two kinds of things:
  - **Parametric memory** — what got compressed into its weights during training. This is a
    *vague, lossy recollection*, frozen at the training cutoff, and you can't see or edit it.
  - **Context memory** — the tokens in the current request (system prompt + conversation +
    anything you pasted/retrieved). This is *exact* and *directly usable*.
- *Build consequence:* If you want reliable, current, or private information used, you must put
  it **in the context** — you cannot rely on the weights. This one fact is the entire reason
  RAG, prompt-stuffing, and tool-use exist. It also means context is finite and precious: every
  token you spend on instructions is a token unavailable for data.

**4. Stochasticity & temperature — the same input gives different outputs.**

*What's actually happening.* Recall that at every step the model outputs a probability for
every possible next token. Suppose after "The capital of France is" the model's top candidates
are:

    " Paris"  → 0.90
    " the"    → 0.04
    " located"→ 0.03
    " a"      → 0.01
    ...(thousands more, each tiny)

The model does **not** simply take " Paris" every time. It runs a weighted random draw — like
rolling a loaded die where " Paris" covers 90% of the faces. Usually you get " Paris," but
occasionally you get " the," and from there the sentence continues down a different path. This
is why two identical requests can produce different answers: each token is a fresh dice roll,
and one early difference cascades into a completely different response.

*Temperature* is the knob that reshapes that die **before** the roll:
  - **Low temperature (→ 0):** sharpen the distribution — the high-probability token gets even
    more dominant. " Paris" might go from 0.90 to effectively 0.99+. Output becomes repetitive
    and near-deterministic. Best when there's a *right* answer.
  - **High temperature (≈ 0.8–1.2):** flatten the distribution — unlikely tokens get a real
    chance. " Paris" might drop toward 0.6 and the alternatives rise. Output becomes more varied
    and creative — and more likely to wander or make mistakes.
  - Think of it as: low temp = "play it safe, pick the obvious word"; high temp = "take risks,
    surprise me."

*Two related knobs you'll see in APIs (top-p and top-k).*
Temperature reshapes the odds, but it never fully removes the long tail of weird, low-
probability tokens — at high temperature those rare tokens can still get picked and derail a
response. **top-p and top-k are guardrails that cut off that tail before sampling.**

Work through one example. Say the next-token candidates are:

    " Paris"     → 0.75
    " France"    → 0.10
    " the"       → 0.06
    " located"   → 0.04
    " Lyon"      → 0.02
    " banana"    → 0.01
    ...(thousands more, each ~0.0001)

  - **top-k — keep a fixed number of candidates.** `top_k = 3` means: throw away everything
    except the 3 most likely tokens, then renormalize and sample among just
    {" Paris", " France", " the"}. " banana" and the long tail are now *impossible*, no matter
    what temperature does. Simple, but rigid — 3 is 3 whether the model is very sure or very
    unsure.
  - **top-p (nucleus sampling) — keep enough candidates to cover probability p.** `top_p = 0.9`
    means: walk down the list adding up probabilities until you reach 0.90, then sample only
    from that set ("the nucleus"). Here 0.75 + 0.10 + 0.06 = 0.91, so the nucleus is
    {" Paris", " France", " the"} and everything else is dropped.
    The key difference from top-k is that the set size **adapts to the model's confidence:**
      - When the model is *confident* (one token at 0.97), the nucleus is just that one token —
        output stays safe.
      - When the model is *uncertain* (probability spread thinly across many tokens), the
        nucleus grows to include more options — output stays diverse.
    This adaptiveness is why top-p is the more popular of the two.

*How they interact with temperature.* Order of operations: temperature reshapes the
distribution first, then top-p/top-k crop it, then a token is sampled from what's left. They
solve different problems: **temperature controls how adventurous the pick is; top-p/top-k cap
how bizarre it's allowed to get.** You can run a creative-but-safe setup (higher temperature
*with* a top-p of ~0.9) — varied wording, but the tail of nonsense tokens still fenced off.

*Determinism caveat (important for engineers).* Even at temperature 0, you are **not
guaranteed** byte-identical output across calls. Providers update models, requests are batched
on GPUs, and floating-point math isn't perfectly reproducible. Some APIs offer a `seed`
parameter for *best-effort* repeatability. Treat "stable output" as something you engineer
toward, never something you assume.

*Practical guidance on the knobs.*
  - Tune **one** primary knob, usually **temperature**; leave top-p at its default (often ~1.0)
    unless you have a reason. Adjusting temperature and top-p together makes results hard to
    reason about.
  - Different providers expose different subsets — OpenAI and Anthropic surface `temperature`
    and `top_p`; `top_k` shows up in some APIs (e.g. Anthropic, many open-weights models) but
    not others. Don't assume all three are available.
  - Defaults are sensible for general use. Reach for top-p/top-k mainly when you see occasional
    "out of nowhere" tokens at higher temperature and want to clamp them without going fully
    deterministic.

*Build consequences (with examples).*
  - **Testing:** A test like `assert reply == "Your order #123 has shipped."` will flake. Test
    the *shape* instead: does it contain the order number? is it valid JSON with the right
    keys? does an LLM-judge rate it as correct? (We cover this properly in [Chapter 7](07-evaluation.md).)
  - **Caching:** If you cache by exact prompt string, tiny differences ("Hello " vs "hello")
    miss the cache. Normalize keys (trim, lowercase) before caching.
  - **Choosing temperature by task:**

        Task                        Suggested temperature
        Data extraction / parsing   0 – 0.2   (you want the same answer every time)
        Classification / routing    0 – 0.3
        Q&A over documents          0.2 – 0.5
        General chat                0.5 – 0.7
        Brainstorming / copywriting 0.8 – 1.0 (you want variety)

  - **Anti-pattern:** don't crank temperature up to "fix" wrong answers. If the model is
    getting facts wrong, more randomness makes it *less* reliable — the fix is better context
    or a better model, not more entropy.

**5. Base vs Instruct/Chat vs Reasoning models — pick the right tool.**

"An LLM" isn't one thing — it's a model captured at a particular post-training stage. The
stages map directly onto what the video showed (pretraining → SFT → RL), and each stage
produces a model that behaves differently and is good for different jobs.

**(a) Base model — the raw autocomplete.**
Output of pretraining only. It has read the internet and will *continue* whatever text you give
it, but it has no concept of "user" and "assistant" and no instinct to be helpful.
  - Give a base model `What is 2 + 2?` and it might reply `What is 3 + 3? What is 4 + 4?` —
    continuing the *pattern* of a worksheet instead of answering, because that's a plausible
    continuation of internet text.
  - Useful for research, raw text completion, and as the starting point for fine-tuning, but
    you essentially never put one in front of end users.

**(b) Instruct / Chat model — the workhorse.**
A base model further trained on example conversations (SFT) and human preferences (RLHF) so it
*follows instructions*, holds a multi-turn dialogue, respects a **system prompt** (a hidden
instruction that sets role/rules/format), and — for most modern ones — can **call tools** and
**return structured output (JSON)**. This is what you call ~95% of the time.
  - Examples: GPT-4-class, Claude Sonnet, Gemini Pro/Flash.
  - Within this tier there's a **capability/price ladder** — and this matters a lot for cost:

        Tier            Rough character                 Typical use
        Small / fast    cheap, quick, "good enough"     classification, routing, extraction,
                                                         high-volume simple calls
        Mid             balanced quality vs cost        most general features, chat, Q&A
        Flagship        best nuance & instruction-      complex generation, tricky
                        following, priciest             instructions, customer-facing quality

    A large app usually uses *several tiers at once* — a cheap model for the 100k simple calls,
    a flagship for the few that need to be excellent.

**(c) Reasoning model — the specialist.**
An instruct model trained further with RL to produce a long internal **chain of thought**
before its final answer (the "thinking" you saw with DeepSeek-R1). It explores, checks itself,
and backtracks — so it's markedly better at problems with many dependent steps.
  - Examples: o3, DeepSeek-R1, Claude with extended thinking.
  - Strong at: competition-grade math, complex/algorithmic code, multi-constraint planning,
    careful logical analysis.
  - The cost: it spends (and you pay for) extra "thinking" tokens, and it's **slower** — often
    seconds to tens of seconds. It's overkill for "What's our return policy?" or "format this
    as JSON," where a chat model answers instantly and just as well.

**How to actually choose (a builder's decision ladder):**
  1. **Start** with a mid-tier chat model and measure quality on *your real task* with *your
     real data* — not on vibes or leaderboards.
  2. **Drop down** to a small/fast/cheap model for the simple, high-volume steps (routing,
     extraction, classification). These are usually most of your call volume, so this is where
     you save the most money and latency.
  3. **Step up** to a flagship chat model only on the steps where output quality visibly
     matters to the user.
  4. **Reach for a reasoning model only** when the task genuinely needs multi-step reasoning,
     and explicitly budget the extra latency and cost.

**Choose for fit, not just "smartness."** Beyond raw capability, the deciding factors are
often: **context-window size** (how much can you feed it?), **tool-calling / structured-output
support**, **multimodal** needs (images, audio), **speed**, **price per token**, **data-
privacy / residency**, and **open-weights vs proprietary** (can you self-host?).

**Worked example — a customer-support app:**
  - Detect the ticket's language and category → *small/fast model*, temperature 0.
  - Pull the answer from your help-center docs and draft a reply → *mid-tier chat model* with
    retrieved context (RAG), low temperature.
  - Handle a gnarly billing dispute with conditional refund rules across multiple policies →
    *reasoning model*, accept the slower response.
  Same app, three model tiers, chosen per task. **The "best" model is the cheapest, fastest one
  that reliably passes your evals — not the top of the leaderboard.**

**6. Hallucination is the default behavior, not an occasional bug.**
The model always produces a *plausible* continuation, whether or not it "knows" the answer — so
it will confidently invent facts, citations, or APIs. Newer models hedge better, but the
tendency is fundamental.
- *Build consequence:* Never trust raw output for anything factual or high-stakes. The
  mitigations — grounding in retrieved context (RAG), giving it tools (search/code), and
  verifying output — are exactly what sections 4–7 teach. Design for verification from day one.

**7. Model families: encoder vs decoder (and where embeddings come from).**

Every chat/instruct/reasoning model above is one architecture family. As a builder you meet a
second family too — usually without realizing it, because it's what powers the embedding model
in your search/RAG stack. The split is about *which tokens each token is allowed to look at*.

**(a) Encoder-only — understand / represent.** *Bidirectional* attention: every token sees the
tokens to its left **and** its right at once. Trained with masked-language-modeling (blank out
a token, predict it from both sides). It does **not** generate text — it reads a whole input
and emits *contextual token embeddings* plus a single pooled vector for the input. Text in →
vector (or label) out.
  - Examples: BERT, RoBERTa, ModernBERT.

**(b) Decoder-only — generate.** *Causal / autoregressive* attention: each token sees only what
came **before** it (it can't peek ahead, because at generation time the future doesn't exist
yet). This is the next-token predictor from concept 1 — text in → more text out. Every chat
model taught elsewhere in this chapter is decoder-only.
  - Examples: GPT, Claude, Gemini.

*One-line mental model:* **encoder = understand/represent** (text in → vector/label);
**decoder = generate** (text in → more text).

*Contextual embeddings, briefly.* "Contextual" means the same word gets a *different* vector
depending on its neighbours — "river *bank*" and "savings *bank*" land in different places in
vector space, because the encoder saw the surrounding words. This is exactly why BERT beat the
older *static* embeddings (word2vec/GloVe), which gave "bank" one fixed vector regardless of
sentence, and it's the reason semantic search actually works: similar *meaning* → nearby
vectors.

*The BERT → RoBERTa → ModernBERT question ("isn't BERT old?").* BERT (2018) is the original
encoder. RoBERTa is the same architecture **trained better** — dropped the next-sentence-
prediction objective, used much more data, and switched to dynamic masking. ModernBERT (2024+)
is a modernized encoder with an 8k context window and heavy code in its training data. So the
*family* is current and actively used even though the first model is old.

*Out of scope (you never need this to build):* the attention math, the MLM/NSP loss internals,
and training any encoder yourself. You call hosted encoders; you don't make them.

- *Build consequence:* (a) The embedding model behind your RAG is an *encoder*-family model and
  your chat model is a *decoder* — you **call** both and **train** neither. (b) Never repurpose
  a decoder's raw hidden states or last-token activations as semantic-search embeddings: a
  decoder only looks left and over-weights the most recent tokens, so its vectors are a poor
  measure of whole-text meaning — use a purpose-built embedding model instead. (c) Picking the
  right embedding model (dimensions, context, language/code coverage) is its own decision; this
  seeds the embedding-model material in the RAG chapter ([Chapter 4](04-rag.md)).

---

#### Resources (beyond the video)
- Re-watch with a builder's eye: segments **1.1–1.6 (0:00–59:28)** and the **hallucination /
  working-memory** part of 1.8 **(1:20:33–1:41:00)**. For each idea, pause and ask: "what does
  this force me to do when I build?"
- **tiktokenizer** — https://tiktokenizer.vercel.app — watch text become tokens in real time.
- One provider's model line-up page (Anthropic's Claude models *or* OpenAI's models page) — map
  "base / chat / reasoning" onto products you'll actually call, and note context-window sizes
  and prices.

---

#### Hands-on (free web UIs, no API keys needed)

**A. Tokens are not words.**
1. Open tiktokenizer. Paste each and record the token count: an English sentence · a code
   snippet · a line of emoji · a non-English phrase · the single word `strawberry`.
2. Note where word ≠ token (capitalization, leading spaces, numbers).
3. Write one sentence: why does "how many r's in strawberry?" trip the model?

**B. The model is stochastic.**
1. In any chat model, ask the same open-ended question 3 times in fresh chats (e.g. "Give me a
   tagline for a coffee shop").
2. Record the variation. If the UI exposes temperature, try it low vs high.
3. Write one sentence on what this means for writing automated tests.

**C. Context = memory (the RAG seed).**
1. Ask a model about an obscure or very recent fact. Observe it hallucinate or hedge.
2. Now paste a paragraph of the real source text into the prompt and ask the same question.
3. Compare the two answers. Write two sentences connecting this to "parametric vs context
   memory" — and predict how a product might fetch that paragraph automatically (you just
   described RAG).

**D. Embeddings are contextual (the encoder vs decoder seed).**
1. Take one ambiguous word in two sentences — "I sat on the river bank" vs "I deposited cash at
   the savings bank" — and send each through a hosted embedding endpoint (any provider's
   embeddings API, or a free embedding playground). You don't train anything; you just call it.

        # tiny shape — embeddings API call (provider-agnostic)
        v1 = embed("I sat on the river bank")          # → vector
        v2 = embed("I deposited cash at the savings bank")
        # the vectors for "bank" differ because the surrounding words differ

2. Confirm the two vectors are not identical (the same word, two meanings → two vectors). That's
   "contextual."
3. Map each model this course uses onto its family: the **chat model** (decoder), the
   **embedding model** (encoder), the **reranker** (encoder). Then write two sentences on why
   you wouldn't use a chat model to produce search vectors.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. In one line, what is an LLM actually doing when it "answers" you?
2. Why is the cost of an API call measured in tokens rather than words or characters?
3. What are the only two sources of knowledge a model can draw on at inference time?
4. Why might the exact same prompt return a different answer on two calls?
5. Why are LLMs bad at counting the letters in a word?
6. What does temperature do to the probability distribution, and when do you want it low?
7. What problem do top-p and top-k solve that temperature alone does not?
8. When would you choose a reasoning model over a standard chat model — and what's the cost?
9. Why is "the model hallucinated" not a surprising failure, given how it works?
10. What does it mean to say the context window is "finite and precious"?
11. What's the difference between an encoder-only and a decoder-only model, in one line each —
    and which family does your chat model belong to, versus your embedding model?

**Apply it (builder scenarios — answer in 2–3 sentences):**
12. A teammate says, "Let's just ask the model for our company's latest refund policy — it was
    probably in the training data." What's wrong with that plan, and what would you do instead?
13. You're building a feature that classifies support tickets into 5 categories. What
    temperature would you lean toward, and which model tier — and why?
14. Your automated test asserts the model's reply equals an exact string, and it keeps failing
    intermittently. Diagnose the root cause and propose a better test.
15. A summarizer works great on famous books but produces vague, wrong summaries for an
    internal PDF. Explain why, using today's concepts, and name the fix.
16. At higher temperature your chatbot occasionally emits a bizarre, off-topic word. How would
    you reduce that *without* making every answer identical?
17. A teammate wants to skip adding an embedding model and just feed your chat model's
    last-token hidden state into the vector database for semantic search. Why is that a bad
    idea, and what should you use instead?

**Stretch / discussion (optional):**
18. If context is the model's working memory, what trade-offs appear as you stuff more and more
    into it? (Think cost, latency, and the model "losing the plot.")
19. Base models are rarely shipped to users. Give one realistic situation where a *builder*
    might still prefer a base model.

**Answer key (peek only after attempting):**
1. Predicting/sampling the next token repeatedly to extend the text plausibly. · 2. The model
processes and is priced per token (its native unit); words/chars don't map 1:1 to compute. ·
3. Parametric memory (weights — vague, frozen) and context memory (the current prompt — exact). ·
4. Outputs are sampled from a probability distribution (temperature > 0). · 5. It sees tokens
(sub-word chunks), not individual characters, and is weak at counting. · 6. It sharpens (low)
or flattens (high) the distribution; want it low when there's a single correct answer
(extraction, classification). · 7. They crop the long tail of unlikely tokens so a rare/bizarre
token can't be sampled; temperature only reshapes odds, it never removes the tail. · 8. For
genuine multi-step reasoning (hard math/code/logic); cost = more latency and money. · 9. It
always emits a *plausible* continuation regardless of whether it knows — confident invention is
built in. · 10. The window has a hard token limit; every instruction token competes with data
tokens, and bigger contexts cost more and can dilute focus. · 11. Encoder-only = bidirectional,
reads a whole input and emits a vector/label (understand/represent); decoder-only = causal,
generates the next token (generate). Your chat model is a decoder; your embedding model is an
encoder. · 12. Weights are a vague, frozen, unverifiable recollection — it'll likely
hallucinate; put the real policy text in the context (retrieve it / RAG). · 13. Low temperature
(0–0.3) and a small/fast tier — you want consistent, cheap, high-volume labels. · 14. Output is
stochastic; exact-match is the wrong assertion — test semantically or with schema/rule
validation. · 15. Famous books are richly represented in the weights (vague recall works); the
internal PDF isn't — supply it in context (RAG). · 16. Add a top-p (~0.9) or top-k guardrail to
fence off the tail while keeping some variety. · 17. A decoder only attends left and over-weights
recent tokens, so its hidden states are a poor representation of whole-text meaning; use a
purpose-built (encoder-family) embedding model. · 18. More tokens = higher cost/latency and risk
of under-weighting buried details ("lost in the middle"). · 19. e.g. raw text completion,
studying model behavior, or specialized fine-tuning/research workflows.

---

**Deliverable:** A short note — 6–8 bullets — titled *"What an LLM is, and what that means for
building on it."* Each bullet = one concept + its practical implication. Attach your answers to
questions 1–17.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~2–3 hours (watching + hands-on + questions).

---

