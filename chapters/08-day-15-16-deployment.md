## Day 15–16 — Deployment & Production: running an LLM feature for real

**Goal:** Take something you built (the Day 7 RAG bot, the Day 9 agent) and stand it up as a real service — one that handles many concurrent users, controls cost and latency *as a system*, ships prompt/model changes without breaking, and stays observable when something fails at 2am. By the end you can answer "what changes when this goes from my laptop to production, and is it ready?"

**Why this matters:** Days 1–14 got you to *"it works on my machine, for one user, when I run it."* Production is a different problem entirely: concurrent users, real money per call, latency people feel, secrets to protect, provider limits to respect, and the need to change the system *without* an outage. None of that shows up in a notebook — it shows up the day real traffic hits. This is the gap between a demo and a feature people depend on. We build *on top of* provider APIs, so this is about **operating a service that calls models**, not running model infrastructure (no GPUs, no self-hosting).

> **Setup assumed:** same as before, plus a prior build to deploy. Code sketches are framework-neutral Python (a generic endpoint, a logging wrapper) — the *patterns* port to FastAPI/Flask/Express/whatever you use. Containers, hosting, and CI are referenced as the surrounding context (run services in Docker, per your infra rules); the depth here is the LLM-specific layer.

**Suggested split:** Day 15 = Parts A–C (production architecture, latency, cost at scale). Day 16 = Parts D–F (reliability/scale under load, shipping changes safely, observability) plus the deliverable.

---

## Part A — The shape of an LLM application in production

**1. The LLM call lives in *your backend*, never in the client.**
The single most important architectural rule: your API key calls the provider from **your server**, and the user's browser/app talks only to *your* backend. The user never holds the key and never calls Anthropic/OpenAI directly. Why: a key in client code is a key on the public internet — anyone can extract it from a web page or mobile app and spend your money. Your backend is also where prompt assembly, RAG, guardrails, logging, and rate-limiting live — none of which you'd trust a client to do.
- *Build consequence:* The shape is always **client → your backend → provider → back**. Your backend is the trust boundary, the control point, and the only thing that ever sees the key. (Key/secret *storage* is out of scope per the program — but "never in the client, never in the repo" is non-negotiable.)

**2. The standard request path — where each earlier skill slots in.**
A production LLM request flows through stages you've already built in isolation:
`client request → authenticate/rate-limit the user → input guardrails (Day 13) → assemble the prompt (Day 5–6) + retrieve context (Day 7) → call the provider with retries (Day 3) → output guardrails (Day 13) → log everything (Part F) → respond.`
Production is largely *wiring these stages into one reliable pipeline* and operating it.
- *Build consequence:* You're not learning new LLM skills here — you're assembling Days 3–14 into a serviceable request handler. When something breaks in prod, it broke in one of these named stages; knowing the pipeline *is* the debugging map.

**3. Sync vs. streaming vs. background — match the delivery pattern to the workload.**
Three patterns, chosen by how long the work takes and how the user waits:
- **Synchronous request/response:** user waits for the full answer. Fine for fast, short calls (a classification, a quick lookup). Simple, but the user stares at a spinner for the whole LLM latency.
- **Streaming** (Day 3 Part B, now an *architecture* choice): stream tokens as generated so the user sees output immediately. The default for anything conversational or long — it transforms *perceived* latency (Part B).
- **Background / async job:** for long-running work (a multi-step agent, batch processing, a big document) — accept the request, return a job id, do the work asynchronously, deliver the result via polling/webhook/push. Don't hold an HTTP connection open for 90 seconds.
- *Build consequence:* Pick the pattern from the *workload*, not habit. A 30-second agent run behind a synchronous endpoint will time out and frustrate users; the same run as a background job with streamed progress feels responsive. The wrong pattern is a production incident waiting to happen.

---

## Part B — Latency: making it feel fast

**4. Measure where the time actually goes before optimizing anything.**
The cardinal rule of performance, applied to LLMs: profile first. In almost every LLM feature, **the provider call dominates** total latency — your code, retrieval, and guardrails are usually milliseconds while the model call is hundreds of ms to seconds. Two numbers to track: **TTFT (time to first token)** — how long until output *starts* (what the user feels first), and **total latency** — until it's *done*. RAG adds an embedding + search hop; agents add one full model call *per step* (Day 10), so their latency is a multiple.
- *Build consequence:* Don't micro-optimize your Python when 95% of the wait is the model. Measure TTFT and total per stage, find the real bottleneck (it's usually "we make too many model calls" or "the model/context is too big"), and aim your effort there.

**5. Streaming is the #1 perceived-latency win — and it's nearly free.**
Streaming doesn't make the answer arrive *sooner* in total — it makes output *start* appearing almost immediately, so the user reads as it generates instead of watching a spinner. A 6-second response that streams *feels* faster than a 3-second one that doesn't. It's the highest-leverage latency improvement because it changes the experience without changing the work.
- *Build consequence:* Stream by default for anything a human reads in real time. The only reason not to is when downstream code needs the *complete* structured output before doing anything (e.g. you must parse full JSON) — then synchronous is honest.

**6. The real latency levers: smaller/faster model, fewer hops, parallelism.**
When streaming isn't enough:
- **Model choice** (Day 2/Day 11): smaller, faster models have lower latency. Use the cheapest/fastest model that passes your eval (Day 13) for the task — often a small model is plenty for classification/extraction, reserving the big model for hard reasoning.
- **Fewer round trips:** every agent step and RAG re-query is serial latency. Cut unnecessary steps; this is *why* Day 9 said "push work down the ladder."
- **Parallelize independent calls:** if you need three independent model/retrieval calls, fire them concurrently, don't await them in sequence — wall-clock becomes the slowest one, not the sum.
- *Build consequence:* Latency is designed in via architecture (model tier, number of hops, concurrency), not patched in afterward. The fastest call is the one you didn't have to make.

---

## Part C — Cost at scale (the bill is now a system property)

**7. The cost model from Day 3, now multiplied by traffic — estimate it explicitly.**
On Day 3 cost was per-call awareness; in production it's a budget line. The math: **(input tokens × input price) + (output tokens × output price), per request, × requests per day.** Do this arithmetic *before* launch. A feature that costs $0.02/request feels free in testing and is **$20,000/month at a million requests.** Agents and RAG multiply it — every agent step and every retrieved chunk is more input tokens (Day 10's super-linear growth).
- *Build consequence:* Produce a real monthly cost estimate at expected traffic *before* shipping, and instrument actual per-request cost in your logs (Part F) so you catch a 3× regression the day a prompt change ships, not on the invoice.

**8. The big cost levers — in order of impact.**
- **Prompt caching** (Day 5–6, now for *cost* not just speed): providers cache a stable prompt *prefix* so repeated calls with the same system prompt / instructions / retrieved context are billed far cheaper on the cached portion. Structure prompts as **stable-prefix → variable-suffix** to maximize hits. Often the single biggest saving for repetitive workloads.
- **Model tiering / routing:** don't send every request to the most expensive model. Route easy requests to a cheap small model and *escalate* only hard ones to the big model (sometimes a cheap model triages and decides). Big savings when most traffic is easy.
- **Shorten the context:** trim/summarize history (Day 3), retrieve fewer/tighter chunks (Day 8 chunking), drop dead weight from prompts. Every token you don't send is money you don't spend, on *every* call.
- **Batching:** for non-urgent bulk work, provider batch APIs offer large discounts in exchange for slower turnaround.
- *Build consequence:* Cost optimization is mostly "send fewer tokens" and "use a cheaper model when you can get away with it" — and your eval set (Day 13) is what tells you when you *can* get away with it. Never trade cost for quality blind; trade it against the number.

**9. Semantic caching — serve a cached answer for an equivalent question.**
Beyond provider prompt caching: keep your *own* cache of answered questions, and when a new question is **semantically equivalent** to a past one (embed it, check cosine similarity against cached questions — Day 7), serve the stored answer with no model call at all. For high-volume features with repetitive questions (support, FAQ), this can deflect a large fraction of traffic.
- *Build consequence:* Powerful but use with judgment — only cache where answers are stable (a refund policy, not "what's my order status"), set a freshness/TTL policy, and tune the similarity threshold so you don't serve a near-miss answer to a subtly different question. A wrong cache hit is a confidently wrong answer.

---

## Part D — Reliability & scale under real traffic

**10. Provider rate limits are the first thing you hit at scale — design for 429s.**
Providers cap you on **RPM (requests/min)** and **TPM (tokens/min)**. Under real concurrency you *will* hit them, and the API returns **429 Too Many Requests** (Day 3 Part E). Naively, that's a user-facing failure. The fixes: **retry with exponential backoff + jitter** (Day 3), a **request queue** with controlled concurrency so you stay under the limit instead of stampeding it, and **backpressure** (shed or queue load gracefully when saturated rather than melting down).
- *Build consequence:* "It worked in testing" with one user says nothing about 200 concurrent users. Rate limits are a capacity-planning input: know your limits, queue to stay under them, and treat 429 as an expected condition to manage, not a crash.

**11. Retries, fallbacks, timeouts — as architecture, not afterthought (recap → extend).**
Day 3 and Day 13 introduced these per-call; in production they become system policy:
- **Retries with backoff** for transient errors (429, 500, 503, timeouts) — but cap them and make them idempotent.
- **Timeouts** on every provider call — never let a hung request hang a user (or hold a worker) forever.
- **Fallbacks:** a *second* path when the primary fails — a different model, a **different provider** (multi-provider failover), a cached/semantic-cache answer, or graceful degradation ("we're busy, try again"). This is the system-level version of Day 13's "reliability is defense in depth."
- *Build consequence:* Assume the provider *will* be slow, rate-limit you, or have an outage — because it will. A production LLM feature has a planned answer for each, so a provider hiccup degrades the experience instead of breaking it.

**12. Keep your backend stateless so you can scale horizontally — state lives elsewhere.**
The API is stateless (Day 3), and your backend should be too: any server instance can handle any request, so you scale by adding instances behind a load balancer. That means **conversation/session state can't live in a server's memory** (it'd vanish on the next request hitting a different instance) — it lives in a shared store (a database/cache, run in Docker). The message history *you* manage (Day 3's "memory is your list") gets persisted and reloaded per request.
- *Build consequence:* Statelessness is what makes scaling easy — but it forces a deliberate decision about *where* conversation state lives. "The bot forgot mid-conversation under load" is almost always session state stuck in one instance's memory instead of a shared store.

---

## Part E — Shipping changes safely (prompts & models are deployable artifacts)

**13. A prompt is a deployable artifact with a version — not a string you edit in place.**
In production, your prompt and your model choice are **configuration you deploy and can roll back**, exactly like code. Each prompt has a version (Day 5–6's templating/versioning), and every logged request records *which* prompt version produced it (Part F). Editing the live prompt by hand, untracked, is the LLM equivalent of editing code directly on the production server.
- *Build consequence:* "Which prompt version is live, what did it score on the eval (Day 13), and can I roll back instantly?" must always have an answer. Treat prompt changes with the same ceremony as code changes — versioned, eval-gated, reversible.

**14. Rollout patterns: gate, canary, A/B, roll back.**
You never flip a prompt/model change to 100% of traffic on a hope:
- **Eval gate (Day 13):** the change must pass your offline eval before it goes anywhere. First line of defense.
- **Canary / gradual rollout:** release to a small % of traffic, watch the live metrics (Part F), then ramp. Limits the blast radius of a bad change.
- **A/B test in prod:** run old vs. new on real traffic and compare real outcomes (user signals, quality, cost) when offline eval can't fully predict reception.
- **Instant rollback:** because prompt/model are versioned config, reverting is flipping back to the previous version — fast, and the reason versioning matters.
- *Build consequence:* Offline eval predicts; production *confirms*. Roll out gradually behind metrics so the inevitable surprise hits 1% of users for ten minutes, not everyone for a day.

**15. Provider models change under you — deprecations and silent updates.**
Unlike your own code, the model is a dependency *someone else* controls. Models get **deprecated** (a version you call is retired — you must migrate) and can be **updated** in place (same name, subtly different behavior). Either can shift your system's behavior without a single change on your side.
- *Build consequence:* Pin specific model versions where you can, watch provider deprecation notices, and — the real safeguard — **re-run your eval set (Day 13) when a model changes**. Your eval suite is exactly what catches "the provider updated the model and our quality dropped" before users do. This is *why* the eval set is permanent infrastructure.

---

## Part F — Observability & operating it

**16. Log every interaction — it's your debugger, your eval-set fuel, and your audit trail.**
For each request, log: the input, the assembled prompt (+ version), the response, **token usage, latency, and cost**, which model, and any guardrail/error events. This one discipline pays off three ways: **debugging** (you can't fix what you can't see — Day 9's tracing, generalized), **eval-set growth** (real failures become eval cases — Day 13's flywheel), and **audit** (what did we tell this user, and why).
```python
# A logging wrapper around the provider call — the backbone of observability.
def tracked_call(messages, *, prompt_version, model="claude-sonnet-4-6"):
    t0 = now()
    resp = call_with_retries(model=model, messages=messages)   # Day 3 retry logic
    log({
        "prompt_version": prompt_version, "model": model,
        "input_tokens": resp.usage.input_tokens, "output_tokens": resp.usage.output_tokens,
        "cost_usd": cost_of(resp.usage, model), "latency_ms": now() - t0,
        "stop_reason": resp.stop_reason,
    })
    return resp
```
- *Build consequence:* Build logging in from request #1, not after the first incident. The team that logs prompt version + tokens + latency + cost per request can answer almost any production question; the team that doesn't is guessing in the dark during an outage.

**17. The production dashboard — the four metrics + traces, watched continuously.**
Surface what Day 13 Part F named: **quality** (online eval / user signals), **latency** (p50 and p95/p99 — the tail is what users feel), **cost** (per-request and total, trending), and **error/refusal rates**. Add **per-feature traces** (the agent's step trajectory, Day 9; which chunks RAG retrieved, Day 7) for drill-down. A spike in any line is a signal; the trace tells you why.
- *Build consequence:* You optimized these in dev; in prod you *monitor* them, because traffic and the model shift under you. An unwatched cost or latency curve is a bill or an outage you'll learn about from users first.

**18. Alerting, incident response, and the feedback loop.**
Monitoring is passive; **alerting** is monitoring that wakes you up — set thresholds on the four metrics (cost spike, p99 latency, error-rate jump, refusal surge) so you find out before users do. When an incident hits, your logs (concept 16) and traces are the investigation. And close the loop: the failures you find in production flow back into the eval set (Day 13's flywheel) so the same bug can't ship twice.
- *Build consequence:* Production isn't "deploy and walk away" — it's a loop: monitor → alert → investigate via logs → fix → add to eval → redeploy. The eval set you launch with is the worst it'll ever be precisely because production keeps feeding it.

**19. Secrets and access at the system level — named, per program scope.**
The key never lives in client code or the repo; it's injected from the environment / a secret manager into your backend only (concept 1). Beyond the provider key: authenticate *your* users, rate-limit *per user* (so one client can't exhaust your provider quota or your budget), and don't log secrets or raw PII (Day 13). Deep secret-management mechanics are out of scope for this program — the principle is what matters: **keys server-side only, per-user limits, no secrets in logs.**
- *Build consequence:* Treat your provider key like a production database password — server-side, injected, never committed, never logged. A leaked key is someone else spending your money; per-user rate limits are what stop one abuser from running up the bill for everyone.

---

**Resources**
- Anthropic & OpenAI — rate-limit docs (RPM/TPM), prompt-caching guides, batch API docs, streaming docs, and model-deprecation/versioning pages.
- A read on **load balancing / horizontal scaling** and **canary/gradual rollouts** at the concept level (general backend ops, applied to an LLM service).
- Your own earlier days: Day 3 (errors/retries/streaming/cost), Day 5–6 (caching/templating), Day 7–8 (RAG hops), Day 9–10 (agent steps/latency), Day 13–14 (metrics, eval gate, reliability) — this section operationalizes all of them.

**Hands-on tasks**
1. **Wrap it in an endpoint:** put your Day 7 RAG bot or Day 9 agent behind a small HTTP endpoint (any framework). Client sends a question, your backend does the work and responds — key stays server-side.
2. **Stream it:** convert the endpoint to stream tokens. Compare the *felt* speed against the synchronous version on a long answer.
3. **Log every call:** add the `tracked_call` wrapper — log prompt version, model, tokens, cost, latency, stop reason — for each request. Print a summary after 10 requests.
4. **Estimate the bill:** from your logs, compute average cost/request, then project monthly cost at 1k, 100k, and 1M requests/day. Write the number down.
5. **Add a cost lever:** apply one — structure the prompt for caching, route easy cases to a smaller model, or trim retrieved context — and measure the per-request cost change against your eval (did quality hold?).
6. **Survive a 429:** simulate rate-limit/transient errors and confirm your retry+backoff+timeout+fallback path degrades gracefully instead of erroring to the user.
7. **Version + roll back:** tag two prompt versions, record which produced each logged response, and demonstrate switching back to the previous version (a config flip, not a code rewrite).
8. *(Stretch)* **Semantic cache:** cache answered questions by embedding; serve a cached answer when a new question is similar enough. Measure the cache-hit rate and name one question you'd *refuse* to cache.

**Questions**

*Check understanding*
1. Why must the provider call happen in your backend and never in the client?
2. Trace the standard production request path and name where Day 7 (RAG) and Day 13 (guardrails) slot in.
3. When do you use synchronous vs. streaming vs. a background job?
4. In a typical LLM feature, what dominates latency, and what are TTFT and total latency?
5. Why does streaming improve *perceived* latency even though total time is unchanged?
6. Give the formula for per-request cost and explain why a "cheap" feature can produce a huge bill.
7. Name three cost levers and one line on how each saves money.
8. What are RPM/TPM, what status code signals you hit them, and what's the response?

*Apply it*
9. Your agent takes ~40 seconds per run. Which delivery pattern fits, and why is a synchronous endpoint wrong?
10. A prompt change passed offline eval. How do you roll it out to avoid a site-wide regression, and how do you undo it if it's bad?
11. Under load, users report the bot "forgets" mid-conversation. What's the likely cause given horizontal scaling?
12. The provider updated the model behind the name you call and quality dropped. What safeguard catches this, and what should you have done to prevent it?
13. Per-request cost tripled the day after a deploy. How does your logging let you find the cause fast?

*Stretch*
14. Design the reliability layer for a high-traffic feature: list every failure mode (429, timeout, 5xx, provider outage) and the architectural response to each.
15. You want to cut cost 50% without hurting quality. Lay out the levers you'd try, in order, and how your eval set tells you when you've gone too far.
16. Explain why prompts and model choices are "deployable artifacts" and what production capabilities (gate, canary, rollback) that framing unlocks.

**Answer key**
1. A key in client code is extractable by anyone (web/mobile) and lets them spend your money; the backend is also where prompt assembly, RAG, guardrails, logging, and rate-limiting must live. Client → backend → provider keeps the key and control server-side.
2. client → authenticate/rate-limit → input guardrails (Day 13) → assemble prompt (Day 5–6) + retrieve (Day 7) → provider call with retries (Day 3) → output guardrails (Day 13) → log → respond. RAG slots in at context assembly; guardrails wrap the model call on both sides.
3. Synchronous for fast short calls; streaming for conversational/long output a human reads in real time; background job for long-running work (multi-step agents, batch) — return a job id, deliver later.
4. The provider call dominates. TTFT = time to first token (when output starts appearing); total latency = until the response is complete.
5. Output starts appearing almost immediately so the user reads as it generates instead of watching a spinner; the experience changes even though total generation time doesn't.
6. (input tokens × input price) + (output tokens × output price), per request × requests/day. A $0.02 request is $20k/month at 1M requests — small per-call cost × large volume = large bill; agents/RAG multiply input tokens.
7. Prompt caching (cheaper billing on a stable cached prefix), model tiering/routing (send easy requests to a cheaper model), shortening context (fewer tokens sent on every call); also batching (bulk discount for slower turnaround).
8. Requests-per-minute and tokens-per-minute caps; hitting them returns 429 Too Many Requests; respond with retry+backoff+jitter, a request queue with controlled concurrency, and backpressure.
9. A background/async job — accept the request, return a job id, deliver via polling/webhook/push (optionally stream progress). A synchronous endpoint would hold the connection ~40s and time out / block a worker.
10. Gate on offline eval, then canary/gradual rollout to a small % while watching the four live metrics, ramp if healthy; roll back instantly by flipping to the previous (versioned) prompt config.
11. Session/conversation state is stuck in one server instance's memory; with horizontal scaling the next request hits a different instance that lacks it. Fix: store session state in a shared store, not instance memory.
12. Re-running your eval set on model change catches the quality drop; prevention = pin specific model versions where possible and watch deprecation/update notices. The eval suite is the safeguard.
13. Logs record per-request tokens, cost, model, and prompt version; compare before/after the deploy to see if input tokens grew (e.g. more retrieved chunks / longer prompt), the model changed, or output ballooned — pinpointing the cause from data, not guesswork.
14. 429 → retry w/ backoff + queue + backpressure; timeout → per-call timeout + retry/fallback; 5xx transient → retry w/ backoff; provider outage → fallback to alternate model/provider, semantic-cache answer, or graceful degradation. Each provider failure has a pre-planned path so the feature degrades, not breaks.
15. Try in order: prompt caching (stable prefix), route easy traffic to a smaller model, trim/summarize context and retrieve fewer chunks, batch non-urgent work, semantic cache for stable repetitive questions. After each, re-run the eval set; if the quality metric drops below your bar, you've gone too far — back off that lever.
16. They're configuration that determines behavior and can be versioned, deployed, and reverted like code; that framing unlocks eval-gating before release, canary/gradual rollout behind metrics, A/B testing in prod, and instant rollback — none of which is possible if the prompt is an untracked string edited in place.

**Deliverable:** wrap a prior build (Day 7 RAG or Day 9 agent) behind a small **API endpoint** that (a) keeps the key server-side, (b) **streams** responses, (c) **logs** prompt version + tokens + latency + cost per request, (d) has **retry + timeout + fallback** on the provider call, and (e) tags responses with a **prompt version** you can roll back. **Plus** a one-page *production-readiness writeup*: estimated monthly cost at a stated traffic level, your latency budget (TTFT target), the rollout + rollback plan, and which failure modes you've handled.

**Daily update (DM to Ayush):** one line — what you deployed/operationalized and any blocker (e.g. "RAG bot behind a streaming endpoint; per-request logging shows ~$0.015/req → ~$450/mo at 1k/day; retry+fallback on 429s; two prompt versions with rollback").

**Time:** ~2 days. Day 15: Parts A–C (architecture, latency, cost) and the endpoint + logging. Day 16: Parts D–F (reliability/scale, safe rollout, observability) and the production-readiness writeup.
