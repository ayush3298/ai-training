# Day 33 — Monitoring, part 1 — drift re-pointed & the improvement loop

> [← Day 32](day-32.md) · [All days](README.md) · [Day 34 →](day-34.md)

**Module:** Monitoring, Drift & the Improvement Loop  ·  **Time:** ~2.5 hrs

## About this module

### Advanced Track — Monitoring, Drift & the Continuous-Improvement Loop: operating the system after you ship

**Goal:** Keep a live LLM feature *good over time* — not just good on launch day. By the end you can re-point classic MLOps "drift / monitoring / retraining" at the artifacts you actually control (you don't own the weights), measure three app-level decays *without* touching a model's parameters, run a fixed retrieval **probe set** on a schedule to catch retrieval rot, and operate the full loop: **trace every request → sample for online eval → let drift signals + thumbs-down surface failures → triage into the eval set → fix the *right* artifact → offline-gate → shadow/canary → roll out or roll back.** You'll walk a real failure (an irrelevant retrieved chunk) from a thumbs-down all the way to a probe-set that prevents its regression.

**Why this matters:** [Chapter 7](07-evaluation.md) gave you the eval set and the metrics; [Chapter 8](08-deployment.md) got you deployed with logging, traces, and a rollout ladder. But a deployed LLM feature is not a finished artifact — it sits in a world that moves under it. Users ask new things, your corpus goes stale, the provider silently ships a new model snapshot, and a prompt edit three weeks ago quietly made answers worse. None of that throws an exception ([Chapter 7](07-evaluation.md)'s *silent failure*, now stretched across months). Classic MLOps answers this with "monitor for drift, then retrain" — but **you don't own the weights**, so "retrain" as gradient updates is almost never the move. The whole chapter is a *re-pointing*: drift becomes decay you can measure from the outside, and "retrain" becomes refresh-the-corpus / revise-the-prompt / re-pin-the-model / retune-retrieval. This is what you *operate* after architecting (the LLM Application Architecture chapter) and deploying ([Chapter 8](08-deployment.md)) — the running-system loop those chapters deferred to here, and the discipline that keeps the Capstone alive past week one.

> **Setup assumed:** same as before, plus a deployed build that already emits **traces with spans** ([Chapter 8](08-deployment.md) Part G — parent request span, child retrieval/inference/tool spans) and **per-request logging** ([Chapter 8](08-deployment.md) Part F). We operate on what those produce; we don't repeat them. Drift math is plain arithmetic over embeddings ([Chapter 4](04-rag.md)) and the same hit-rate@k / validated-judge instruments you built on [Chapter 7](07-evaluation.md). No GPUs, no weight access — everything here works *on top of* the provider API. Run any backend (Langfuse/Phoenix) in Docker, per your infra rules.

---

## Part A — Re-pointing "drift": three app-level decays you can measure without the weights

**1. Classic drift assumes you own the model — you don't, so re-point it.**
In classic MLOps, "data drift" (input distribution moves) and "model drift / concept drift" (the learned mapping goes stale) are diagnosed by watching feature distributions and a model whose internals and training data you control. Building *on top of* an LLM API, two of those assumptions are false: you can't inspect the weights, and the provider can change them under you ([Chapter 8](08-deployment.md) concept 15). So the question isn't "did my model's parameters drift?" — it's **"did the behavior of the *system I assembled* decay, and can I detect it from the outside?"** The answer is yes, via three things you *can* observe: the inputs arriving, the chunks your retriever returns, and the outputs leaving.
- *Build consequence:* Don't go looking for weight drift you can't see. Re-point the entire drift apparatus at the three signals below — every one is computable from data you already log/trace ([Chapter 8](08-deployment.md)), with no model access.

**2. Decay #1 — input-distribution shift (the live queries stop looking like your baseline).**
The closest analogue to classic data drift, and the only one that maps cleanly. **Embed your live queries** ([Chapter 4](04-rag.md)'s embedding model), and watch the **centroid** of a rolling window move away from a fixed **baseline window** (e.g. your first stable month of traffic). Concretely: take the mean embedding vector of the baseline queries and of the current window, and track the distance between them (cosine or Euclidean); optionally cluster (e.g. HDBSCAN over the embeddings) and watch new clusters appear or old ones empty out. A growing distance / a fresh cluster means users are now asking things your system was never tuned or evaluated for.
- *Build consequence:* Input shift doesn't *break* anything by itself — it's an early-warning that your eval set and corpus may no longer represent reality. When the centroid drifts, the *correct* response is usually "sample these new queries into the eval set" ([Part C](#part-c--the-full-continuous-improvement-loop-built-on-the-eval--deployment-grafts)), not "panic." It tells you *where* to look, not *that* something is wrong.

**3. Decay #2 — retrieval-quality decay (the build-on-top-specific one, and the one to watch hardest).**
This has no clean classic analogue because the retriever is *your* component sitting between the user and a frozen model — and it's where most build-on-top systems rot first (stale corpus, an embedding-model change, a query distribution the index never saw). Measure it two ways, both cheap:
- **Continuous mean cosine of query↔retrieved-doc:** for each live request, you already have (from the retrieval span, [Chapter 8](08-deployment.md)) the query embedding and the top-k chunk embeddings. Track the **mean** top-1 (or mean top-k) query↔chunk cosine over a rolling window. When it drops **>2σ below the baseline-window mean**, retrieval is returning worse matches than it used to — the index is no longer answering today's questions well.
- **A fixed retrieval PROBE SET, re-run on a schedule.** Curate **50–100** `(query, known-correct-chunk-id)` items over your corpus and compute **hit-rate@k** on them ([Chapter 7](07-evaluation.md) concept 11) — *the exact eval-chapter metric, now run continuously* instead of once. Re-run nightly/weekly (or on every corpus reindex). A fixed probe set isolates the *retriever*: because the queries never change, any drop in hit-rate@k is your retrieval pipeline degrading, not the questions getting harder.
- *Build consequence:* The probe set is your retrieval smoke alarm — the single highest-leverage monitor for a RAG system you operate. The >2σ cosine watch catches *live* degradation continuously; the probe set hit-rate@k catches it *deterministically and attributably*. Run both; alert on either.

**4. Decay #3 — output-quality decay (the answers get worse, silently).**
The end-to-end symptom, caught from signals on the output side:
- **Sampled online-judge faithfulness trending down** — sample live answers, score them with your *validated* faithfulness judge ([Chapter 7](07-evaluation.md) concepts 14–17), attach the score to the trace ([Chapter 8](08-deployment.md)), and watch the rolling average. A downward trend = answers drifting off their grounding.
- **Refusal / "I don't know" rate rising** — the model increasingly abstaining is itself a decay signal (often a retrieval miss starving it of context, or a prompt change).
- **Thumbs-down rate climbing** — the cheapest real-user signal (the [Chapter 7](07-evaluation.md) feedback-loop graft); a rising rate is a decay alarm even before you know *why*.
- **The provider silently shipped a new model snapshot** — same model name, different behavior ([Chapter 8](08-deployment.md) concept 15). This often manifests as a *step change* in one of the above on a date you didn't deploy — the giveaway that the change came from *outside* your code.
- *Build consequence:* These are lagging indicators (the harm already reached a user) where decays #1–#2 are leading ones — which is exactly why you watch all three. A step change in output quality with **no deploy of yours** on that date is the fingerprint of a provider model update; re-run your eval set ([Chapter 7](07-evaluation.md)) to size it and decide whether to re-pin.

---

## Part B — Re-pointing "retraining": for build-on-top, "retrain" = refresh the artifacts you control

**5. You almost never run gradient updates — you refresh artifacts instead.**
Classic MLOps responds to drift by **retraining**: collect fresh labels, run gradient descent, ship new weights. Building on top of a provider model, you have no weights to update (fine-tuning exists — [Chapter 6](06-customization.md) — but it's rare, expensive, and *not* your first move). So "retrain" re-points to **refresh the artifacts you actually control**, in rough order of how often you'll reach for each:
- **Refresh / re-chunk / re-embed the RAG corpus** — add new docs, drop stale ones, re-chunk, regenerate embeddings ([Chapter 4](04-rag.md)). The most common "retrain" for a RAG system.
- **Revise the prompt** — the system prompt, instructions, format ([Chapter 3](03-prompt-engineering.md)). Cheapest and fastest; also the most common *source* of regressions ([Chapter 8](08-deployment.md) concept 23).
- **Update the few-shots** — swap in better in-context examples for the failures you're seeing ([Chapter 3](03-prompt-engineering.md)).
- **Re-pin or migrate the provider model** — move off a deprecated/updated snapshot, or to a better/cheaper one ([Chapter 8](08-deployment.md) concept 15).
- **Retune retrieval** — k, similarity thresholds, add/swap a reranker, metadata filters ([Chapter 7](07-evaluation.md) retrieval metrics tell you if it helped).
- **Add the failure to the eval set** — so the fix is gated and can't silently regress ([Chapter 7](07-evaluation.md) concept 19). This one is *not optional* — it accompanies every fix above.
- *Build consequence:* When a decay signal fires, the engineering question is **"which artifact do I refresh?"**, never "do I retrain the model?" [Part D](#part-d--a-worked-path-handle-irrelevant-chunk-retrieval) walks a concrete diagnosis → artifact mapping. Picking the *wrong* artifact (re-prompting when retrieval is the problem) is the most common wasted-effort failure here.

**6. Cadence is event-driven, not calendar-driven.**
Classic retraining often runs on a schedule (weekly, monthly) because data drifts continuously and predictably. Here the *trigger* is a **signal**, not a date: a probe-set hit-rate drop, a >2σ cosine dip, a thumbs-down cluster, a rising refusal rate, or a provider model-update notice. You refresh the relevant artifact *when a signal says so* — which is why [Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights)'s monitors exist at all. (The one thing you *do* run on a schedule is the *monitoring*: the nightly/weekly probe-set run, the rolling-window stats — the detectors are scheduled, the *response* is triggered.)
- *Build consequence:* Don't put "re-embed the corpus" on a monthly cron and call it monitoring — that re-embeds for no reason and still misses a mid-month provider snapshot. Schedule the *detectors*; trigger the *fix* off what they report. Event-driven response is cheaper and catches the surprises a calendar can't.

---

## Part C — The full continuous-improvement loop (built on the eval + deployment grafts)

**7. The loop, end to end — and which earlier piece each stage reuses.**
Everything above assembles into one operating cycle. Each stage is a thing you already built; this Part is the *wiring*, not new machinery:
`trace every request (Chapter 8 Part G) → sample for online eval — LLM-judge + user thumbs, scores attached to the trace (Chapter 7 judge + the feedback-loop graft) → drift signals (Part A) + thumbs-down surface failures → TRIAGE the failures into the eval set (Chapter 7 concept 19) → fix the RIGHT artifact (Part B) → offline gate on the eval set (Chapter 7/8 eval gate) → shadow / canary (Chapter 8 Part G shadow + the rollout ladder) → roll out or roll back (Chapter 8 concept 14).`
Then it repeats: the rolled-out change is itself traced, sampled, and watched. The eval set you launched with is the *worst* it'll ever be ([Chapter 7](07-evaluation.md) concept 27 / [Chapter 8](08-deployment.md) concept 18) — this loop is *why*.
- *Build consequence:* Production is not "deploy and walk away" and it's not a one-off "monitor" bolt-on — it's this closed cycle, and every arrow is an artifact from [Chapter 7](07-evaluation.md) or [Chapter 8](08-deployment.md). If any arrow is missing (no traces → can't sample; no judge → can't score; no eval gate → can't ship safely; no probe set → can't see retrieval rot), the loop is open and decay reaches users uncaught.

**8. Sampling for online eval — you can't judge every request, so sample deliberately.**
At volume, LLM-judging 100% of traffic is too slow and too expensive (every judge call is a model call — [Chapter 7](07-evaluation.md) concept 14). So **sample**: judge a random fraction (e.g. 1–5%) for an unbiased quality trend, **plus** *always* judge the requests that already carry a negative user signal (thumbs-down, an escalation, a refusal) — those are your richest failure source and you want every one. The judge scores get **attached to the trace as scores** (Langfuse/Phoenix model this natively as a `score` object on a trace/observation), so a trace carries both *what happened* (spans) and *how good it was* (scores) in one place.
- *Build consequence:* Random sampling gives you the honest trend line ([Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) decay #3); targeted sampling of negative-signal requests gives you the triage queue (concept 9). Do both — the random sample tells you *whether* quality is moving, the targeted sample tells you *what to fix*.

**9. Triage — turn surfaced failures into eval cases (the loop's hinge).**
Drift signals and thumbs-down produce a *pile of suspect requests*; triage is the discipline of turning that pile into **eval cases and a diagnosis**. For each surfaced failure: open its trace ([Chapter 8](08-deployment.md) Part G), read the spans to find *where* it went wrong (retrieval returned junk? generation hallucinated? wrong query understanding?), label the correct expected output, and **add it to the eval set** ([Chapter 7](07-evaluation.md) concept 19) — *before* you fix it, so the eval can confirm the fix and catch a future regression. This is the same flywheel [Chapter 7](07-evaluation.md)/8 named, now driven by the *monitoring* signals of [Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) rather than ad-hoc bug reports.
- *Build consequence:* Triage is where monitoring becomes improvement — without it, dashboards just tell you you're getting worse. The trace makes triage fast (you read *which span* failed instead of guessing); the eval-set append makes the fix *permanent*. Skipping the append is how the same failure ships twice.

---

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
