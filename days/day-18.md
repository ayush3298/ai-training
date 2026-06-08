# Day 18 — Deployment, part 1 — architecture, latency & cost

> [← Day 17](day-17.md) · [All days](README.md) · [Day 19 →](day-19.md)

**Module:** Deployment & Production  ·  **Time:** ~2.5–3 hrs

## About this module

### Chapter 8 — Deployment & Production: running an LLM feature for real

**Goal:** Take something you built (the [Chapter 4](04-rag.md) RAG bot, the [Chapter 5](05-agents.md) agent) and stand it up as a real service — one that handles many concurrent users, controls cost and latency *as a system*, ships prompt/model changes without breaking, and stays observable when something fails at 2am. By the end you can answer "what changes when this goes from my laptop to production, and is it ready?"

**Why this matters:** Chapters 1–7 got you to *"it works on my machine, for one user, when I run it."* Production is a different problem entirely: concurrent users, real money per call, latency people feel, secrets to protect, provider limits to respect, and the need to change the system *without* an outage. None of that shows up in a notebook — it shows up the day real traffic hits. This is the gap between a demo and a feature people depend on. We build *on top of* provider APIs, so this is about **operating a service that calls models**, not running model infrastructure (no GPUs, no self-hosting).

> **Setup assumed:** same as before, plus a prior build to deploy. Code sketches are framework-neutral Python (a generic endpoint, a logging wrapper) — the *patterns* port to FastAPI/Flask/Express/whatever you use. Containers, hosting, and CI are referenced as the surrounding context (run services in Docker, per your infra rules); the depth here is the LLM-specific layer.

---

## Part A — The shape of an LLM application in production

**1. The LLM call lives in *your backend*, never in the client.**
The single most important architectural rule: your API key calls the provider from **your server**, and the user's browser/app talks only to *your* backend. The user never holds the key and never calls Anthropic/OpenAI directly. Why: a key in client code is a key on the public internet — anyone can extract it from a web page or mobile app and spend your money. Your backend is also where prompt assembly, RAG, guardrails, logging, and rate-limiting live — none of which you'd trust a client to do.
- *Build consequence:* The shape is always **client → your backend → provider → back**. Your backend is the trust boundary, the control point, and the only thing that ever sees the key. (Key/secret *storage* is out of scope per the program — but "never in the client, never in the repo" is non-negotiable.)

**2. The standard request path — where each earlier skill slots in.**
A production LLM request flows through stages you've already built in isolation:
`client request → authenticate/rate-limit the user → input guardrails (Chapter 7) → assemble the prompt (Chapter 3) + retrieve context (Chapter 4) → call the provider with retries (Chapter 2) → output guardrails (Chapter 7) → log everything (Part F) → respond.`
Production is largely *wiring these stages into one reliable pipeline* and operating it.
- *Build consequence:* You're not learning new LLM skills here — you're assembling Chapters 2–7 into a serviceable request handler. When something breaks in prod, it broke in one of these named stages; knowing the pipeline *is* the debugging map.

**3. Sync vs. streaming vs. background — match the delivery pattern to the workload.**
Three patterns, chosen by how long the work takes and how the user waits:
- **Synchronous request/response:** user waits for the full answer. Fine for fast, short calls (a classification, a quick lookup). Simple, but the user stares at a spinner for the whole LLM latency.
- **Streaming** ([Chapter 2](02-apis-and-integration.md) Part B, now an *architecture* choice): stream tokens as generated so the user sees output immediately. The default for anything conversational or long — it transforms *perceived* latency ([Part B](#part-b--latency-making-it-feel-fast)).
- **Background / async job:** for long-running work (a multi-step agent, batch processing, a big document) — accept the request, return a job id, do the work asynchronously, deliver the result via polling/webhook/push. Don't hold an HTTP connection open for 90 seconds.
- *Build consequence:* Pick the pattern from the *workload*, not habit. A 30-second agent run behind a synchronous endpoint will time out and frustrate users; the same run as a background job with streamed progress feels responsive. The wrong pattern is a production incident waiting to happen.

---

## Part B — Latency: making it feel fast

**4. Measure where the time actually goes before optimizing anything.**
The cardinal rule of performance, applied to LLMs: profile first. In almost every LLM feature, **the provider call dominates** total latency — your code, retrieval, and guardrails are usually milliseconds while the model call is hundreds of ms to seconds. Two numbers to track: **TTFT (time to first token)** — how long until output *starts* (what the user feels first), and **total latency** — until it's *done*. RAG adds an embedding + search hop; agents add one full model call *per step* ([Chapter 5](05-agents.md)), so their latency is a multiple.
- *Build consequence:* Don't micro-optimize your Python when 95% of the wait is the model. Measure TTFT and total per stage, find the real bottleneck (it's usually "we make too many model calls" or "the model/context is too big"), and aim your effort there.

**5. Streaming is the #1 perceived-latency win — and it's nearly free.**
Streaming doesn't make the answer arrive *sooner* in total — it makes output *start* appearing almost immediately, so the user reads as it generates instead of watching a spinner. A 6-second response that streams *feels* faster than a 3-second one that doesn't. It's the highest-leverage latency improvement because it changes the experience without changing the work.
- *Build consequence:* Stream by default for anything a human reads in real time. The only reason not to is when downstream code needs the *complete* structured output before doing anything (e.g. you must parse full JSON) — then synchronous is honest.

**6. The real latency levers: smaller/faster model, fewer hops, parallelism.**
When streaming isn't enough:
- **Model choice** ([Chapter 1](01-foundations.md)/[Chapter 6](06-customization.md)): smaller, faster models have lower latency. Use the cheapest/fastest model that passes your eval ([Chapter 7](07-evaluation.md)) for the task — often a small model is plenty for classification/extraction, reserving the big model for hard reasoning.
- **Fewer round trips:** every agent step and RAG re-query is serial latency. Cut unnecessary steps; this is *why* [Chapter 5](05-agents.md) said "push work down the ladder."
- **Parallelize independent calls:** if you need three independent model/retrieval calls, fire them concurrently, don't await them in sequence — wall-clock becomes the slowest one, not the sum.
- *Build consequence:* Latency is designed in via architecture (model tier, number of hops, concurrency), not patched in afterward. The fastest call is the one you didn't have to make.

---

## Part C — Cost at scale (the bill is now a system property)

**7. The cost model from [Chapter 2](02-apis-and-integration.md), now multiplied by traffic — estimate it explicitly.**
On [Chapter 2](02-apis-and-integration.md) cost was per-call awareness; in production it's a budget line. The math: **(input tokens × input price) + (output tokens × output price), per request, × requests per day.** Do this arithmetic *before* launch. A feature that costs $0.02/request feels free in testing and is **$20,000/month at a million requests.** Agents and RAG multiply it — every agent step and every retrieved chunk is more input tokens ([Chapter 5](05-agents.md)'s super-linear growth).
- *Build consequence:* Produce a real monthly cost estimate at expected traffic *before* shipping, and instrument actual per-request cost in your logs ([Part F](#part-f--observability--operating-it)) so you catch a 3× regression the day a prompt change ships, not on the invoice.

**8. The big cost levers — in order of impact.**
- **Prompt caching** ([Chapter 3](03-prompt-engineering.md), now for *cost* not just speed): providers cache a stable prompt *prefix* so repeated calls with the same system prompt / instructions / retrieved context are billed far cheaper on the cached portion. Structure prompts as **stable-prefix → variable-suffix** to maximize hits. Often the single biggest saving for repetitive workloads.
- **Model tiering / routing:** don't send every request to the most expensive model. Route easy requests to a cheap small model and *escalate* only hard ones to the big model (sometimes a cheap model triages and decides). Big savings when most traffic is easy.
- **Shorten the context:** trim/summarize history ([Chapter 2](02-apis-and-integration.md)), retrieve fewer/tighter chunks ([Chapter 4](04-rag.md) chunking), drop dead weight from prompts. Every token you don't send is money you don't spend, on *every* call.
- **Batching:** for non-urgent bulk work, provider batch APIs offer large discounts in exchange for slower turnaround.
- *Build consequence:* Cost optimization is mostly "send fewer tokens" and "use a cheaper model when you can get away with it" — and your eval set ([Chapter 7](07-evaluation.md)) is what tells you when you *can* get away with it. Never trade cost for quality blind; trade it against the number.

**9. Semantic caching — serve a cached answer for an equivalent question.**
Beyond provider prompt caching: keep your *own* cache of answered questions, and when a new question is **semantically equivalent** to a past one (embed it, check cosine similarity against cached questions — [Chapter 4](04-rag.md)), serve the stored answer with no model call at all. For high-volume features with repetitive questions (support, FAQ), this can deflect a large fraction of traffic.
- *Build consequence:* Powerful but use with judgment — only cache where answers are stable (a refund policy, not "what's my order status"), set a freshness/TTL policy, and tune the similarity threshold so you don't serve a near-miss answer to a subtly different question. A wrong cache hit is a confidently wrong answer.

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. How can you optimize model latency?
2. How do you reduce LLM latency at scale?
3. How do you monitor an LLM in production?
4. What are the model deployment approaches?
5. How do you containerize and deploy a model?
6. How do you monitor ML models in production?
7. Can you deploy your own model in AWS Bedrock?
8. What is model deployment in machine learning?
9. How do you handle rollback of a deployed model?
10. How do you handle latency in LLM-based systems?
11. Which AWS services have you used in production?
12. How do you manage and deploy LLMs in production?
13. How do you integrate Azure OpenAI in production?
14. How do you deploy a model on custom GPU servers?
15. How do you run tests for ML models in production?

_(66 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
