## Advanced Track — Monitoring, Drift & the Continuous-Improvement Loop: operating the system after you ship

**Goal:** Keep a live LLM feature *good over time* — not just good on launch day. By the end you can re-point classic MLOps "drift / monitoring / retraining" at the artifacts you actually control (you don't own the weights), measure three app-level decays *without* touching a model's parameters, run a fixed retrieval **probe set** on a schedule to catch retrieval rot, and operate the full loop: **trace every request → sample for online eval → let drift signals + thumbs-down surface failures → triage into the eval set → fix the *right* artifact → offline-gate → shadow/canary → roll out or roll back.** You'll walk a real failure (an irrelevant retrieved chunk) from a thumbs-down all the way to a probe-set that prevents its regression.

**Why this matters:** Chapter 7 gave you the eval set and the metrics; Chapter 8 got you deployed with logging, traces, and a rollout ladder. But a deployed LLM feature is not a finished artifact — it sits in a world that moves under it. Users ask new things, your corpus goes stale, the provider silently ships a new model snapshot, and a prompt edit three weeks ago quietly made answers worse. None of that throws an exception (Chapter 7's *silent failure*, now stretched across months). Classic MLOps answers this with "monitor for drift, then retrain" — but **you don't own the weights**, so "retrain" as gradient updates is almost never the move. The whole chapter is a *re-pointing*: drift becomes decay you can measure from the outside, and "retrain" becomes refresh-the-corpus / revise-the-prompt / re-pin-the-model / retune-retrieval. This is what you *operate* after architecting (the LLM Application Architecture chapter) and deploying (Chapter 8) — the running-system loop those chapters deferred to here, and the discipline that keeps the Capstone alive past week one.

> **Setup assumed:** same as before, plus a deployed build that already emits **traces with spans** (Chapter 8 Part G — parent request span, child retrieval/inference/tool spans) and **per-request logging** (Chapter 8 Part F). We operate on what those produce; we don't repeat them. Drift math is plain arithmetic over embeddings (Chapter 4) and the same hit-rate@k / validated-judge instruments you built on Chapter 7. No GPUs, no weight access — everything here works *on top of* the provider API. Run any backend (Langfuse/Phoenix) in Docker, per your infra rules.

**Suggested split:** Session 1 = Parts A–C (re-point drift to three app-level decays; re-point "retraining" to artifact refresh; the full continuous-improvement loop wired onto the eval + deployment grafts); Session 2 = Parts D–F (the worked irrelevant-chunk path; production-grade RAG ops + agentic challenges; cross-team consistency + the 2026 tooling), plus the deliverable.

---

## Part A — Re-pointing "drift": three app-level decays you can measure without the weights

**1. Classic drift assumes you own the model — you don't, so re-point it.**
In classic MLOps, "data drift" (input distribution moves) and "model drift / concept drift" (the learned mapping goes stale) are diagnosed by watching feature distributions and a model whose internals and training data you control. Building *on top of* an LLM API, two of those assumptions are false: you can't inspect the weights, and the provider can change them under you (Chapter 8 concept 15). So the question isn't "did my model's parameters drift?" — it's **"did the behavior of the *system I assembled* decay, and can I detect it from the outside?"** The answer is yes, via three things you *can* observe: the inputs arriving, the chunks your retriever returns, and the outputs leaving.
- *Build consequence:* Don't go looking for weight drift you can't see. Re-point the entire drift apparatus at the three signals below — every one is computable from data you already log/trace (Chapter 8), with no model access.

**2. Decay #1 — input-distribution shift (the live queries stop looking like your baseline).**
The closest analogue to classic data drift, and the only one that maps cleanly. **Embed your live queries** (Chapter 4's embedding model), and watch the **centroid** of a rolling window move away from a fixed **baseline window** (e.g. your first stable month of traffic). Concretely: take the mean embedding vector of the baseline queries and of the current window, and track the distance between them (cosine or Euclidean); optionally cluster (e.g. HDBSCAN over the embeddings) and watch new clusters appear or old ones empty out. A growing distance / a fresh cluster means users are now asking things your system was never tuned or evaluated for.
- *Build consequence:* Input shift doesn't *break* anything by itself — it's an early-warning that your eval set and corpus may no longer represent reality. When the centroid drifts, the *correct* response is usually "sample these new queries into the eval set" (Part C), not "panic." It tells you *where* to look, not *that* something is wrong.

**3. Decay #2 — retrieval-quality decay (the build-on-top-specific one, and the one to watch hardest).**
This has no clean classic analogue because the retriever is *your* component sitting between the user and a frozen model — and it's where most build-on-top systems rot first (stale corpus, an embedding-model change, a query distribution the index never saw). Measure it two ways, both cheap:
- **Continuous mean cosine of query↔retrieved-doc:** for each live request, you already have (from the retrieval span, Chapter 8) the query embedding and the top-k chunk embeddings. Track the **mean** top-1 (or mean top-k) query↔chunk cosine over a rolling window. When it drops **>2σ below the baseline-window mean**, retrieval is returning worse matches than it used to — the index is no longer answering today's questions well.
- **A fixed retrieval PROBE SET, re-run on a schedule.** Curate **50–100** `(query, known-correct-chunk-id)` items over your corpus and compute **hit-rate@k** on them (Chapter 7 concept 11) — *the exact eval-chapter metric, now run continuously* instead of once. Re-run nightly/weekly (or on every corpus reindex). A fixed probe set isolates the *retriever*: because the queries never change, any drop in hit-rate@k is your retrieval pipeline degrading, not the questions getting harder.
- *Build consequence:* The probe set is your retrieval smoke alarm — the single highest-leverage monitor for a RAG system you operate. The >2σ cosine watch catches *live* degradation continuously; the probe set hit-rate@k catches it *deterministically and attributably*. Run both; alert on either.

**4. Decay #3 — output-quality decay (the answers get worse, silently).**
The end-to-end symptom, caught from signals on the output side:
- **Sampled online-judge faithfulness trending down** — sample live answers, score them with your *validated* faithfulness judge (Chapter 7 concepts 14–17), attach the score to the trace (Chapter 8), and watch the rolling average. A downward trend = answers drifting off their grounding.
- **Refusal / "I don't know" rate rising** — the model increasingly abstaining is itself a decay signal (often a retrieval miss starving it of context, or a prompt change).
- **Thumbs-down rate climbing** — the cheapest real-user signal (the Chapter 7 feedback-loop graft); a rising rate is a decay alarm even before you know *why*.
- **The provider silently shipped a new model snapshot** — same model name, different behavior (Chapter 8 concept 15). This often manifests as a *step change* in one of the above on a date you didn't deploy — the giveaway that the change came from *outside* your code.
- *Build consequence:* These are lagging indicators (the harm already reached a user) where decays #1–#2 are leading ones — which is exactly why you watch all three. A step change in output quality with **no deploy of yours** on that date is the fingerprint of a provider model update; re-run your eval set (Chapter 7) to size it and decide whether to re-pin.

---

## Part B — Re-pointing "retraining": for build-on-top, "retrain" = refresh the artifacts you control

**5. You almost never run gradient updates — you refresh artifacts instead.**
Classic MLOps responds to drift by **retraining**: collect fresh labels, run gradient descent, ship new weights. Building on top of a provider model, you have no weights to update (fine-tuning exists — Chapter 6 — but it's rare, expensive, and *not* your first move). So "retrain" re-points to **refresh the artifacts you actually control**, in rough order of how often you'll reach for each:
- **Refresh / re-chunk / re-embed the RAG corpus** — add new docs, drop stale ones, re-chunk, regenerate embeddings (Chapter 4). The most common "retrain" for a RAG system.
- **Revise the prompt** — the system prompt, instructions, format (Chapter 3). Cheapest and fastest; also the most common *source* of regressions (Chapter 8 concept 23).
- **Update the few-shots** — swap in better in-context examples for the failures you're seeing (Chapter 3).
- **Re-pin or migrate the provider model** — move off a deprecated/updated snapshot, or to a better/cheaper one (Chapter 8 concept 15).
- **Retune retrieval** — k, similarity thresholds, add/swap a reranker, metadata filters (Chapter 7 retrieval metrics tell you if it helped).
- **Add the failure to the eval set** — so the fix is gated and can't silently regress (Chapter 7 concept 19). This one is *not optional* — it accompanies every fix above.
- *Build consequence:* When a decay signal fires, the engineering question is **"which artifact do I refresh?"**, never "do I retrain the model?" Part D walks a concrete diagnosis → artifact mapping. Picking the *wrong* artifact (re-prompting when retrieval is the problem) is the most common wasted-effort failure here.

**6. Cadence is event-driven, not calendar-driven.**
Classic retraining often runs on a schedule (weekly, monthly) because data drifts continuously and predictably. Here the *trigger* is a **signal**, not a date: a probe-set hit-rate drop, a >2σ cosine dip, a thumbs-down cluster, a rising refusal rate, or a provider model-update notice. You refresh the relevant artifact *when a signal says so* — which is why Part A's monitors exist at all. (The one thing you *do* run on a schedule is the *monitoring*: the nightly/weekly probe-set run, the rolling-window stats — the detectors are scheduled, the *response* is triggered.)
- *Build consequence:* Don't put "re-embed the corpus" on a monthly cron and call it monitoring — that re-embeds for no reason and still misses a mid-month provider snapshot. Schedule the *detectors*; trigger the *fix* off what they report. Event-driven response is cheaper and catches the surprises a calendar can't.

---

## Part C — The full continuous-improvement loop (built on the eval + deployment grafts)

**7. The loop, end to end — and which earlier piece each stage reuses.**
Everything above assembles into one operating cycle. Each stage is a thing you already built; this Part is the *wiring*, not new machinery:
`trace every request (Chapter 8 Part G) → sample for online eval — LLM-judge + user thumbs, scores attached to the trace (Chapter 7 judge + the feedback-loop graft) → drift signals (Part A) + thumbs-down surface failures → TRIAGE the failures into the eval set (Chapter 7 concept 19) → fix the RIGHT artifact (Part B) → offline gate on the eval set (Chapter 7/8 eval gate) → shadow / canary (Chapter 8 Part G shadow + the rollout ladder) → roll out or roll back (Chapter 8 concept 14).`
Then it repeats: the rolled-out change is itself traced, sampled, and watched. The eval set you launched with is the *worst* it'll ever be (Chapter 7 concept 27 / Chapter 8 concept 18) — this loop is *why*.
- *Build consequence:* Production is not "deploy and walk away" and it's not a one-off "monitor" bolt-on — it's this closed cycle, and every arrow is an artifact from Chapter 7 or Chapter 8. If any arrow is missing (no traces → can't sample; no judge → can't score; no eval gate → can't ship safely; no probe set → can't see retrieval rot), the loop is open and decay reaches users uncaught.

**8. Sampling for online eval — you can't judge every request, so sample deliberately.**
At volume, LLM-judging 100% of traffic is too slow and too expensive (every judge call is a model call — Chapter 7 concept 14). So **sample**: judge a random fraction (e.g. 1–5%) for an unbiased quality trend, **plus** *always* judge the requests that already carry a negative user signal (thumbs-down, an escalation, a refusal) — those are your richest failure source and you want every one. The judge scores get **attached to the trace as scores** (Langfuse/Phoenix model this natively as a `score` object on a trace/observation), so a trace carries both *what happened* (spans) and *how good it was* (scores) in one place.
- *Build consequence:* Random sampling gives you the honest trend line (Part A decay #3); targeted sampling of negative-signal requests gives you the triage queue (concept 9). Do both — the random sample tells you *whether* quality is moving, the targeted sample tells you *what to fix*.

**9. Triage — turn surfaced failures into eval cases (the loop's hinge).**
Drift signals and thumbs-down produce a *pile of suspect requests*; triage is the discipline of turning that pile into **eval cases and a diagnosis**. For each surfaced failure: open its trace (Chapter 8 Part G), read the spans to find *where* it went wrong (retrieval returned junk? generation hallucinated? wrong query understanding?), label the correct expected output, and **add it to the eval set** (Chapter 7 concept 19) — *before* you fix it, so the eval can confirm the fix and catch a future regression. This is the same flywheel Chapter 7/8 named, now driven by the *monitoring* signals of Part A rather than ad-hoc bug reports.
- *Build consequence:* Triage is where monitoring becomes improvement — without it, dashboards just tell you you're getting worse. The trace makes triage fast (you read *which span* failed instead of guessing); the eval-set append makes the fix *permanent*. Skipping the append is how the same failure ships twice.

---

## Part D — A worked path: "handle irrelevant chunk retrieval"

**10. From a single thumbs-down to a regression-proof fix — the loop in one concrete trace.**
This is the loop of Part C run on the most common RAG complaint. Walk it stage by stage:
1. **Signal:** a user hits **thumbs-down** on an answer (the Chapter 7 feedback-loop graft captured it; it's attached to the trace).
2. **Open its TRACE** (Chapter 8 Part G): the parent request span with its child spans — input guardrail, **retrieval span**, inference span.
3. **Read the retrieval span:** it returned an **off-topic chunk** in the top-k — the query was about *refund eligibility window*, the top chunk was about *shipping times*. The generator then either hallucinated or answered the wrong question off that junk context.
4. **Diagnose:** this is **retrieval-quality decay** (Part A decay #2), *not* a generation bug — the generator did its best with bad context. (Chapter 7's whole point: the four RAG dimensions / the retrieval-vs-generation split localize the broken component. Low context precision/recall here, not low faithfulness-given-good-context.)
5. **Fix the RIGHT artifact** (Part B) — one or more of: **re-chunk** (the relevant passage was split so it never embedded as a clean unit), **add a metadata filter** (scope retrieval to the `refunds` section), **add a reranker** (push the on-topic chunk above the off-topic one), or **refresh the corpus** (the refund-policy doc was missing/stale).
6. **Add the query to the retrieval PROBE SET** (Part A concept 3) with its known-correct chunk id — so its hit-rate@k is now monitored and **it can't silently regress**. Also add the full case to the eval set (Part C concept 9).
7. **Gate → shadow/canary → roll out** (Chapter 8): confirm the probe-set hit-rate@k *recovers* and the eval doesn't regress elsewhere, then ship behind the ladder.
- *Build consequence:* The thumbs-down didn't just get a one-off patch — it became a *permanent probe-set item*, so the system is now monitored against exactly that failure forever. This is the difference between firefighting and operating: every fix tightens the net. The trace is what made step 4 a *diagnosis* instead of a guess.

---

## Part E — Production-grade RAG operations & agentic challenges

**11. Corpus freshness is an operational property, not a one-time load.**
The corpus you indexed on launch day starts going stale immediately — policies change, docs get added, products ship. A RAG system whose corpus isn't refreshed answers today's questions from yesterday's facts, and the symptom is *rising refusal/IDK rate* or *falling faithfulness* (Part A decay #3) with no code change on your side. Treat ingestion/re-chunk/re-embed as a **pipeline you operate** (the LLM Application Architecture chapter's data-plane framing), with a freshness policy: how often you re-ingest, how you detect changed source docs, how you retire stale ones.
- *Build consequence:* "The bot gave outdated info" is usually a corpus-freshness failure, not a model failure — and no prompt tweak fixes it. Own the refresh cadence (event-driven on source changes where you can — Part B concept 6), and let the probe set + faithfulness trend tell you when freshness has slipped.

**12. When the embedding model changes, you MUST reindex/re-embed — the whole corpus.**
A subtle, high-blast-radius one: query embeddings and stored chunk embeddings must come from the **same** embedding model, or cosine similarity is meaningless (you'd be comparing vectors from two different spaces). So if you upgrade or the provider changes the embedding model, **every stored chunk must be re-embedded** and the index rebuilt — a partial migration silently destroys retrieval quality, and the probe-set hit-rate@k will crater. (This is exactly the reindex/migration trigger the RAG vector-DB graft flags.)
- *Build consequence:* Pin your embedding-model version like you pin your generation model (Chapter 8 concept 15). Changing it is a full reindex event, gated and verified against the probe set *before* it serves traffic — never a hot-swap. The off-topic-chunk fix of Part D is the *retail* version of this; an embedding-model change is the *wholesale* version.

**13. Agentic systems break the snapshot-and-replay assumption.**
Single-shot RAG has a clean trace you can replay. **Agents** (Chapter 5) take **non-deterministic execution paths** — different runs of the same input take different numbers of steps, different tool calls, different branches — so you *cannot* snapshot one run and replay it as "the" behavior the way you can for a fixed pipeline. Three operational consequences: (a) you monitor *trajectories* and per-step quality, not just final answers; (b) **trace volume grows fast** — past roughly **1k runs/day**, a multi-step agent emits tens of thousands of spans/day, and ad-hoc log reading stops scaling (you need a trace backend with search/filter — Part F); (c) **per-step trace inspection is the only viable debugger** — when an agent goes wrong, you open the trace and find *which step* (which tool call, which retrieval, which inference) broke, because there's no deterministic path to re-run.
- *Build consequence:* Budget for trace volume *before* an agent ships — a backend that searches/filters/samples spans (Part F) is mandatory past ~1k runs/day, not a nice-to-have. And accept that the trace *is* the debugger for agents (Chapter 8 concept 20, now at scale): you can't reproduce non-determinism, so you must *observe* it.

---

## Part F — Cross-team consistency & the 2026 tooling

**14. Output consistency across teams = shared judge + shared eval set + versioned prompts as source of truth.**
When multiple engineers/teams touch the same feature (or several features share a model), quality fragments unless three things are *shared and authoritative*:
- **One shared, validated judge** (Chapter 7 concept 17) — so "good" means the same thing to everyone, and two teams' numbers are comparable instead of each grading to its own private rubric.
- **One shared eval set** — the common bar everything gates against, growing from everyone's triaged failures (Part C), so a fix by one team can't silently regress another's cases.
- **Versioned prompts as the source of truth** (Chapter 3 / Chapter 8 concept 13) — the prompt that's live is the one in version control with a known eval score, not a string someone edited in a console.
- *Build consequence:* Without these three, "it works for my team" recreates Chapter 7's *vibes* problem at the organizational scale — divergent judges and private eval sets mean nobody can compare results or trust a cross-team change. The shared judge + shared eval set + versioned prompts are the org-level version of the eval discipline.

**15. The 2026 tooling where this loop is already wired — adopt observability, not an orchestration framework.**
You do *not* need to rewrite onto an agent framework to get this loop (Chapter 8 concept 22's point, extended to monitoring). These tools wire tracing + online eval + scores + drift, and read the vendor-neutral standard:
- **OpenTelemetry GenAI semantic conventions** — the vendor-neutral span schema (Chapter 8 concept 21); emit these so any backend below can read your traces. Still maturing as of 2026 — pin your instrumentation version.
- **Langfuse** — open-source, self-hostable in Docker; traces + `score` objects + LLM-as-judge online evaluators on production traces. The common default for owning your data.
- **Arize Phoenix** — open-source; strong on **embedding/drift analysis** (UMAP projection, HDBSCAN clustering, drift vs a reference baseline — exactly Part A decay #1) alongside tracing and evals.
- **LangSmith** — managed, rich trace UI + dataset/eval tooling.
- **Helicone** — drop-in proxy (one base-URL change) for logging/tracing with near-zero instrumentation.
- *Build consequence:* Start cheap (a proxy, or self-hosted Langfuse/Phoenix in Docker) and instrument to the **OTel GenAI** standard so you're not locked to a vendor. The loop is a *layer you add* over the Chapter 8 architecture — you're choosing where spans and scores land and which tool computes the drift, not migrating your application onto a framework.

---

**Resources**
- **OpenTelemetry — GenAI semantic conventions:** the spans spec — https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/ ; overview — https://opentelemetry.io/docs/specs/semconv/gen-ai/ ; agent/framework spans — https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/
- **Langfuse — evaluation & scores:** overview — https://langfuse.com/docs/evaluation/overview ; scores data model — https://langfuse.com/docs/evaluation/scores/data-model ; LLM-as-a-judge (incl. online eval on production traces) — https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge
- **Arize Phoenix — embeddings & drift analysis:** embeddings analysis cookbook — https://arize.com/docs/phoenix/cookbook/retrieval-and-inferences/embeddings-analysis ; embedding-drift method (reference vs current baseline) — https://docs.arize.com/arize/machine-learning/computer-vision/how-to-cv/embedding-drift ; repo — https://github.com/Arize-ai/phoenix
- **RAGAS — RAG metrics (faithfulness, context precision/recall) run continuously:** available metrics — https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/ ; concepts — https://docs.ragas.io/en/v0.1.21/concepts/metrics/
- **Drift-detection methods for LLM apps:** the cosine-baseline + probe-set / fixed-eval-on-a-schedule pattern — read Phoenix's embedding-drift writeup (above) for the centroid/cluster method, and pair it with Chapter 7's hit-rate@k as the scheduled probe.
- **Your own earlier chapters — this chapter operates all of them:** Chapter 4 (embeddings, chunking, RAG metrics), Chapter 5 (agent trajectories), Chapter 7 (eval set, hit-rate@k/MRR, the validated judge, the feedback-loop graft), Chapter 8 (Part F logging, Part G traces/spans + OTel + shadow/canary rollout ladder), and the LLM Application Architecture chapter (compound-system + data-plane framing). This is what you *operate* after architecting and deploying.

**Hands-on tasks** *(Docker-first — run any trace/eval backend in a container per your infra rules)*
1. **Build a 50-item retrieval probe set** over the course corpus: `(query, known-correct-chunk-id)` pairs spanning your corpus's topics. Compute and record the **baseline hit-rate@k** (k=4) — the same metric as Chapter 7 concept 11, now saved as a baseline to compare against.
2. **Seed a regression and detect it:** degrade retrieval deliberately — *either* swap/garble a handful of corpus docs *or* change the embedding model and re-embed (Part E concept 12) — re-run the probe set, and show the **hit-rate@k drop** vs baseline. State which decay this is (Part A decay #2) and how the probe set attributed it to retrieval.
3. **Walk the thumbs-down → fix → recover path (Part D):** take one probe item now failing, **open/construct its trace**, identify the **off-topic chunk** in the retrieval span, apply **one** fix (re-chunk / metadata filter / reranker / refresh the doc), re-run the probe set, and show hit-rate@k **recover**. Add that query to the probe set so it's now guarded.
4. **Attach an online-judge score to a trace:** sample a few live/replayed answers, run your Chapter 7 **validated faithfulness judge**, and attach the score to each trace (a `score` object if using Langfuse/Phoenix). Show a rolling faithfulness average you could trend (Part A decay #3).
5. **Wire the targeted-sampling rule (concept 8):** always-judge any request carrying a thumbs-down/refusal; random-sample the rest. Show the two queues (trend sample vs triage queue).
6. *(Stretch — input-distribution drift)* **Compute query-embedding centroid drift** between two query batches (a "baseline" batch and a "today" batch): embed both, take each batch's mean vector, report the cosine/Euclidean distance between centroids, and state whether it indicates input shift (Part A decay #1) worth sampling into the eval set.

**Questions**

*Check understanding*
1. Why doesn't classic "model/data drift + retrain" transfer directly to a system you build *on top of* an LLM API?
2. Name the three app-level decays and the one observable signal each is measured from.
3. What is a retrieval **probe set**, what metric do you run on it, and why does keeping the queries *fixed* matter?
4. For build-on-top systems, list four things "retrain" actually re-points to. Why is a gradient update almost never the first move?
5. Why is the improvement cadence **event-driven** rather than calendar-driven — and what *is* run on a schedule?
6. Trace the full continuous-improvement loop and name which earlier chapter supplies each stage.
7. Why can't you LLM-judge 100% of production traffic, and what two-part sampling rule fixes it?

*Apply it*
8. A user thumbs-downs an answer. Walk the trace-driven path to a regression-proof fix, and say which decay it is and which artifact you'd refresh.
9. Output-quality metrics step-change downward on a date you shipped *nothing*. What's the likely cause and how do you confirm + respond?
10. Your mean query↔retrieved-doc cosine has dropped 2.5σ below baseline over two weeks, with no deploy. Name two plausible causes and what you'd check first.
11. You're about to upgrade the embedding model. What must you do to the corpus before serving traffic, and what proves it worked?
12. An agent feature is at ~5k runs/day and "ad-hoc log reading isn't working." What changed at scale, and what do you need?

*Stretch*
13. Two teams share one model but each keeps a private rubric and eval set. What goes wrong, and what three shared artifacts fix it?
14. Design the monitoring for a RAG support bot: which detectors run on a schedule, which signals trigger a fix, and how a surfaced failure becomes a permanent guard.
15. Distinguish input-distribution drift from retrieval-quality decay: how each is measured, why a fix for one doesn't fix the other, and which is the leading vs lagging indicator.

**Answer key**
1. You don't own the weights — you can't inspect them and the provider can change them under you — so "retrain via gradient descent" is mostly unavailable. Re-point: measure decay in the *system you assembled* (inputs, retrieval, outputs) from the outside, and "retrain" by refreshing the artifacts you control.
2. (a) Input-distribution shift — embed live queries, watch the centroid/clusters move vs a baseline window; (b) retrieval-quality decay — mean query↔retrieved-doc cosine dropping >2σ from baseline *and* a fixed probe set's scheduled hit-rate@k; (c) output-quality decay — sampled online-judge faithfulness trend, refusal/IDK rate, thumbs-down rate (and a provider snapshot showing as a step change).
3. A fixed 50–100 item set of `(query, known-correct-chunk-id)` pairs over your corpus; run **hit-rate@k** on a schedule. Fixed queries isolate the retriever — any hit-rate drop is the pipeline degrading, not the questions getting harder.
4. Refresh/re-chunk/re-embed the corpus; revise the prompt; update few-shots; re-pin/migrate the provider model; retune retrieval (k/thresholds/reranker); add the failure to the eval set. A gradient update (fine-tune) is rare, expensive, needs no weight access you have, and usually the wrong/most-expensive lever — corpus/prompt/retrieval fixes solve most decays faster.
5. Drift arrives unpredictably (a provider snapshot, a corpus going stale, a query-mix shift), so you respond to a *signal*, not a date; a calendar re-embed wastes work and still misses mid-cycle surprises. What runs on a schedule is the *detectors* — the nightly/weekly probe-set run and the rolling-window stats.
6. trace every request (Chapter 8 Part G) → sample for online eval, judge + thumbs, scores on the trace (Chapter 7 judge + feedback-loop graft) → drift signals (Part A) + thumbs-down surface failures → triage into the eval set (Chapter 7 concept 19) → fix the right artifact (Part B) → offline gate (Chapter 7/8) → shadow/canary (Chapter 8 Part G + ladder) → roll out or roll back (Chapter 8 concept 14).
7. Every judge call is a model call — judging all traffic is too slow/expensive. Fix: random-sample a small fraction (1–5%) for an unbiased quality trend, **and** always judge every request carrying a negative signal (thumbs-down/refusal/escalation) for the triage queue.
8. Open the trace → read the retrieval span → find the off-topic chunk → diagnose **retrieval-quality decay** (not generation) → refresh the right artifact (re-chunk / metadata filter / reranker / refresh corpus) → add the query to the probe set (and the case to the eval set) so it's guarded → gate/shadow/canary → roll out, confirming probe-set hit-rate@k recovered.
9. A provider silently shipped a new model snapshot (same name, changed behavior) — the fingerprint is a step change with no deploy of yours. Confirm by re-running your eval set (Chapter 7) to size the regression; respond by re-pinning to a prior version if available, or revising prompt/retrieval to adapt, gated as usual.
10. Likely retrieval-quality decay: the corpus went stale relative to a shifted query mix, or the embedding model changed/was changed. Check first: run the probe set (does hit-rate@k confirm the drop and attribute it to retrieval?), then check for an embedding-model or corpus change, and sample the drifted live queries.
11. Re-embed the **entire** corpus with the new model and rebuild the index (query and chunk embeddings must share one model/space) — never partial. Prove it by running the probe-set hit-rate@k against baseline *before* serving, and gate/canary the reindex.
12. Non-deterministic agent paths can't be snapshot-and-replayed, and trace volume grew past where ad-hoc reading scales (~1k+ runs/day → tens of thousands of spans). You need a trace backend that searches/filters/samples spans, and you debug by per-step trace inspection, not log scrolling.
13. Quality fragments — divergent judges mean numbers aren't comparable, private eval sets mean one team's fix can silently regress another's cases (Chapter 7 vibes at org scale). Fix: one shared validated judge, one shared growing eval set, and versioned prompts in source control as the single source of truth.
14. Detectors on a schedule: nightly probe-set hit-rate@k, rolling query↔doc cosine, sampled faithfulness trend, refusal/thumbs-down rates, input-centroid drift. Triggers: any probe-set drop / >2σ cosine dip / thumbs-down cluster / refusal spike / provider-update notice fires a fix. A surfaced failure → open trace → triage into eval set → fix artifact → add the query to the probe set so it's permanently guarded → gate/canary.
15. Input-distribution drift = the *questions* change (measured by query-embedding centroid/cluster movement vs a baseline window) — a *leading* indicator that says "sample these into the eval set." Retrieval-quality decay = the *retriever returns worse chunks* (measured by query↔doc cosine >2σ drop and probe-set hit-rate@k). Different fixes: input shift → expand eval set / possibly grow the corpus; retrieval decay → re-chunk/re-embed/retune/rerank. Output-quality decay is the *lagging* indicator (harm already shipped); input shift is the earliest leading one.

**Deliverable:** a **monitoring + improvement-loop kit** for a prior build (Chapter 4 RAG bot or Chapter 5 agent): (a) a **≥50-item retrieval probe set** with a recorded **baseline hit-rate@k**; (b) a demonstrated **seeded regression** (garbled corpus *or* embedding-model change) showing the hit-rate@k drop and its attribution to retrieval-quality decay; (c) the **worked irrelevant-chunk path** (Part D) — a thumbs-down/probe failure traced to an off-topic chunk, **one** fix applied, and the probe-set hit-rate@k shown to **recover**, with that query added to the probe set; (d) at least one **online-judge faithfulness score attached to a trace** with a rolling trend; and (e) a one-paragraph **operating note**: which detectors you'd schedule, which signals trigger which artifact refresh (your Part B mapping), and how a surfaced failure becomes a permanent guard. *(Optional: the query-centroid drift number between two batches.)*

**Daily update:** one line — what you monitored/closed the loop on and any blocker (e.g. "monitoring kit on the RAG bot: 50-item probe set, baseline hit-rate@4 0.86; garbled-corpus regression dropped it to 0.62; traced a thumbs-down to an off-topic chunk, added a metadata filter → recovered to 0.88 and added the query to the probe set; faithfulness judge scores now attached to traces").

**Time:** ~2 sessions. Session 1: Parts A–C (re-point drift to the three decays; re-point "retraining" to artifact refresh; wire the full loop onto the Chapter 7 eval + Chapter 8 deployment grafts) and the probe set + baseline + regression. Session 2: Parts D–F (the worked irrelevant-chunk path, production RAG ops + agentic challenges, cross-team consistency + 2026 tooling) and the monitoring-loop deliverable.
