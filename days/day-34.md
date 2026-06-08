# Day 34 — Monitoring, part 2 — worked path, RAG ops & 2026 tooling

> [← Day 33](day-33.md) · [All days](README.md) · [Day 35 →](day-35.md)

**Module:** Monitoring, Drift & the Improvement Loop  ·  **Time:** ~2.5 hrs

## Where we are

_Continues **Monitoring, Drift & the Improvement Loop**. Earlier days covered Parts A, B, C; today picks up where they left off._

---

## Part D — A worked path: "handle irrelevant chunk retrieval"

**10. From a single thumbs-down to a regression-proof fix — the loop in one concrete trace.**
This is the loop of [Part C](#part-c--the-full-continuous-improvement-loop-built-on-the-eval--deployment-grafts) run on the most common RAG complaint. Walk it stage by stage:
1. **Signal:** a user hits **thumbs-down** on an answer (the [Chapter 7](07-evaluation.md) feedback-loop graft captured it; it's attached to the trace).
2. **Open its TRACE** ([Chapter 8](08-deployment.md) Part G): the parent request span with its child spans — input guardrail, **retrieval span**, inference span.
3. **Read the retrieval span:** it returned an **off-topic chunk** in the top-k — the query was about *refund eligibility window*, the top chunk was about *shipping times*. The generator then either hallucinated or answered the wrong question off that junk context.
4. **Diagnose:** this is **retrieval-quality decay** ([Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) decay #2), *not* a generation bug — the generator did its best with bad context. ([Chapter 7](07-evaluation.md)'s whole point: the four RAG dimensions / the retrieval-vs-generation split localize the broken component. Low context precision/recall here, not low faithfulness-given-good-context.)
5. **Fix the RIGHT artifact** ([Part B](#part-b--re-pointing-retraining-for-build-on-top-retrain--refresh-the-artifacts-you-control)) — one or more of: **re-chunk** (the relevant passage was split so it never embedded as a clean unit), **add a metadata filter** (scope retrieval to the `refunds` section), **add a reranker** (push the on-topic chunk above the off-topic one), or **refresh the corpus** (the refund-policy doc was missing/stale).
6. **Add the query to the retrieval PROBE SET** ([Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) concept 3) with its known-correct chunk id — so its hit-rate@k is now monitored and **it can't silently regress**. Also add the full case to the eval set ([Part C](#part-c--the-full-continuous-improvement-loop-built-on-the-eval--deployment-grafts) concept 9).
7. **Gate → shadow/canary → roll out** ([Chapter 8](08-deployment.md)): confirm the probe-set hit-rate@k *recovers* and the eval doesn't regress elsewhere, then ship behind the ladder.
- *Build consequence:* The thumbs-down didn't just get a one-off patch — it became a *permanent probe-set item*, so the system is now monitored against exactly that failure forever. This is the difference between firefighting and operating: every fix tightens the net. The trace is what made step 4 a *diagnosis* instead of a guess.

---

## Part E — Production-grade RAG operations & agentic challenges

**11. Corpus freshness is an operational property, not a one-time load.**
The corpus you indexed on launch day starts going stale immediately — policies change, docs get added, products ship. A RAG system whose corpus isn't refreshed answers today's questions from yesterday's facts, and the symptom is *rising refusal/IDK rate* or *falling faithfulness* ([Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) decay #3) with no code change on your side. Treat ingestion/re-chunk/re-embed as a **pipeline you operate** (the LLM Application Architecture chapter's data-plane framing), with a freshness policy: how often you re-ingest, how you detect changed source docs, how you retire stale ones.
- *Build consequence:* "The bot gave outdated info" is usually a corpus-freshness failure, not a model failure — and no prompt tweak fixes it. Own the refresh cadence (event-driven on source changes where you can — [Part B](#part-b--re-pointing-retraining-for-build-on-top-retrain--refresh-the-artifacts-you-control) concept 6), and let the probe set + faithfulness trend tell you when freshness has slipped.

**12. When the embedding model changes, you MUST reindex/re-embed — the whole corpus.**
A subtle, high-blast-radius one: query embeddings and stored chunk embeddings must come from the **same** embedding model, or cosine similarity is meaningless (you'd be comparing vectors from two different spaces). So if you upgrade or the provider changes the embedding model, **every stored chunk must be re-embedded** and the index rebuilt — a partial migration silently destroys retrieval quality, and the probe-set hit-rate@k will crater. (This is exactly the reindex/migration trigger the RAG vector-DB graft flags.)
- *Build consequence:* Pin your embedding-model version like you pin your generation model ([Chapter 8](08-deployment.md) concept 15). Changing it is a full reindex event, gated and verified against the probe set *before* it serves traffic — never a hot-swap. The off-topic-chunk fix of [Part D](#part-d--a-worked-path-handle-irrelevant-chunk-retrieval) is the *retail* version of this; an embedding-model change is the *wholesale* version.

**13. Agentic systems break the snapshot-and-replay assumption.**
Single-shot RAG has a clean trace you can replay. **Agents** ([Chapter 5](05-agents.md)) take **non-deterministic execution paths** — different runs of the same input take different numbers of steps, different tool calls, different branches — so you *cannot* snapshot one run and replay it as "the" behavior the way you can for a fixed pipeline. Three operational consequences: (a) you monitor *trajectories* and per-step quality, not just final answers; (b) **trace volume grows fast** — past roughly **1k runs/day**, a multi-step agent emits tens of thousands of spans/day, and ad-hoc log reading stops scaling (you need a trace backend with search/filter — [Part F](#part-f--cross-team-consistency--the-2026-tooling)); (c) **per-step trace inspection is the only viable debugger** — when an agent goes wrong, you open the trace and find *which step* (which tool call, which retrieval, which inference) broke, because there's no deterministic path to re-run.
- *Build consequence:* Budget for trace volume *before* an agent ships — a backend that searches/filters/samples spans ([Part F](#part-f--cross-team-consistency--the-2026-tooling)) is mandatory past ~1k runs/day, not a nice-to-have. And accept that the trace *is* the debugger for agents ([Chapter 8](08-deployment.md) concept 20, now at scale): you can't reproduce non-determinism, so you must *observe* it.

---

## Part F — Cross-team consistency & the 2026 tooling

**14. Output consistency across teams = shared judge + shared eval set + versioned prompts as source of truth.**
When multiple engineers/teams touch the same feature (or several features share a model), quality fragments unless three things are *shared and authoritative*:
- **One shared, validated judge** ([Chapter 7](07-evaluation.md) concept 17) — so "good" means the same thing to everyone, and two teams' numbers are comparable instead of each grading to its own private rubric.
- **One shared eval set** — the common bar everything gates against, growing from everyone's triaged failures ([Part C](#part-c--the-full-continuous-improvement-loop-built-on-the-eval--deployment-grafts)), so a fix by one team can't silently regress another's cases.
- **Versioned prompts as the source of truth** ([Chapter 3](03-prompt-engineering.md) / [Chapter 8](08-deployment.md) concept 13) — the prompt that's live is the one in version control with a known eval score, not a string someone edited in a console.
- *Build consequence:* Without these three, "it works for my team" recreates [Chapter 7](07-evaluation.md)'s *vibes* problem at the organizational scale — divergent judges and private eval sets mean nobody can compare results or trust a cross-team change. The shared judge + shared eval set + versioned prompts are the org-level version of the eval discipline.

**15. The 2026 tooling where this loop is already wired — adopt observability, not an orchestration framework.**
You do *not* need to rewrite onto an agent framework to get this loop ([Chapter 8](08-deployment.md) concept 22's point, extended to monitoring). These tools wire tracing + online eval + scores + drift, and read the vendor-neutral standard:
- **OpenTelemetry GenAI semantic conventions** — the vendor-neutral span schema ([Chapter 8](08-deployment.md) concept 21); emit these so any backend below can read your traces. Still maturing as of 2026 — pin your instrumentation version.
- **Langfuse** — open-source, self-hostable in Docker; traces + `score` objects + LLM-as-judge online evaluators on production traces. The common default for owning your data.
- **Arize Phoenix** — open-source; strong on **embedding/drift analysis** (UMAP projection, HDBSCAN clustering, drift vs a reference baseline — exactly [Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) decay #1) alongside tracing and evals.
- **LangSmith** — managed, rich trace UI + dataset/eval tooling.
- **Helicone** — drop-in proxy (one base-URL change) for logging/tracing with near-zero instrumentation.
- *Build consequence:* Start cheap (a proxy, or self-hosted Langfuse/Phoenix in Docker) and instrument to the **OTel GenAI** standard so you're not locked to a vendor. The loop is a *layer you add* over the [Chapter 8](08-deployment.md) architecture — you're choosing where spans and scores land and which tool computes the drift, not migrating your application onto a framework.

---

---

## Module wrap-up — hands-on, questions & deliverable

**Resources**
- **OpenTelemetry — GenAI semantic conventions:** the spans spec — https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/ ; overview — https://opentelemetry.io/docs/specs/semconv/gen-ai/ ; agent/framework spans — https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/
- **Langfuse — evaluation & scores:** overview — https://langfuse.com/docs/evaluation/overview ; scores data model — https://langfuse.com/docs/evaluation/scores/data-model ; LLM-as-a-judge (incl. online eval on production traces) — https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge
- **Arize Phoenix — embeddings & drift analysis:** embeddings analysis cookbook — https://arize.com/docs/phoenix/cookbook/retrieval-and-inferences/embeddings-analysis ; embedding-drift method (reference vs current baseline) — https://docs.arize.com/arize/machine-learning/computer-vision/how-to-cv/embedding-drift ; repo — https://github.com/Arize-ai/phoenix
- **RAGAS — RAG metrics (faithfulness, context precision/recall) run continuously:** available metrics — https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/ ; concepts — https://docs.ragas.io/en/v0.1.21/concepts/metrics/
- **Drift-detection methods for LLM apps:** the cosine-baseline + probe-set / fixed-eval-on-a-schedule pattern — read Phoenix's embedding-drift writeup (above) for the centroid/cluster method, and pair it with [Chapter 7](07-evaluation.md)'s hit-rate@k as the scheduled probe.
- **Your own earlier chapters — this chapter operates all of them:** [Chapter 4](04-rag.md) (embeddings, chunking, RAG metrics), [Chapter 5](05-agents.md) (agent trajectories), [Chapter 7](07-evaluation.md) (eval set, hit-rate@k/MRR, the validated judge, the feedback-loop graft), [Chapter 8](08-deployment.md) (Part F logging, Part G traces/spans + OTel + shadow/canary rollout ladder), and the LLM Application Architecture chapter (compound-system + data-plane framing). This is what you *operate* after architecting and deploying.

**Hands-on tasks** *(Docker-first — run any trace/eval backend in a container per your infra rules)*
1. **Build a 50-item retrieval probe set** over the course corpus: `(query, known-correct-chunk-id)` pairs spanning your corpus's topics. Compute and record the **baseline hit-rate@k** (k=4) — the same metric as [Chapter 7](07-evaluation.md) concept 11, now saved as a baseline to compare against.
2. **Seed a regression and detect it:** degrade retrieval deliberately — *either* swap/garble a handful of corpus docs *or* change the embedding model and re-embed ([Part E](#part-e--production-grade-rag-operations--agentic-challenges) concept 12) — re-run the probe set, and show the **hit-rate@k drop** vs baseline. State which decay this is ([Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) decay #2) and how the probe set attributed it to retrieval.
3. **Walk the thumbs-down → fix → recover path ([Part D](#part-d--a-worked-path-handle-irrelevant-chunk-retrieval)):** take one probe item now failing, **open/construct its trace**, identify the **off-topic chunk** in the retrieval span, apply **one** fix (re-chunk / metadata filter / reranker / refresh the doc), re-run the probe set, and show hit-rate@k **recover**. Add that query to the probe set so it's now guarded.
4. **Attach an online-judge score to a trace:** sample a few live/replayed answers, run your [Chapter 7](07-evaluation.md) **validated faithfulness judge**, and attach the score to each trace (a `score` object if using Langfuse/Phoenix). Show a rolling faithfulness average you could trend ([Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) decay #3).
5. **Wire the targeted-sampling rule (concept 8):** always-judge any request carrying a thumbs-down/refusal; random-sample the rest. Show the two queues (trend sample vs triage queue).
6. *(Stretch — input-distribution drift)* **Compute query-embedding centroid drift** between two query batches (a "baseline" batch and a "today" batch): embed both, take each batch's mean vector, report the cosine/Euclidean distance between centroids, and state whether it indicates input shift ([Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights) decay #1) worth sampling into the eval set.

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
6. trace every request ([Chapter 8](08-deployment.md) Part G) → sample for online eval, judge + thumbs, scores on the trace ([Chapter 7](07-evaluation.md) judge + feedback-loop graft) → drift signals ([Part A](#part-a--re-pointing-drift-three-app-level-decays-you-can-measure-without-the-weights)) + thumbs-down surface failures → triage into the eval set ([Chapter 7](07-evaluation.md) concept 19) → fix the right artifact ([Part B](#part-b--re-pointing-retraining-for-build-on-top-retrain--refresh-the-artifacts-you-control)) → offline gate ([Chapter 7](07-evaluation.md)/8) → shadow/canary ([Chapter 8](08-deployment.md) Part G + ladder) → roll out or roll back ([Chapter 8](08-deployment.md) concept 14).
7. Every judge call is a model call — judging all traffic is too slow/expensive. Fix: random-sample a small fraction (1–5%) for an unbiased quality trend, **and** always judge every request carrying a negative signal (thumbs-down/refusal/escalation) for the triage queue.
8. Open the trace → read the retrieval span → find the off-topic chunk → diagnose **retrieval-quality decay** (not generation) → refresh the right artifact (re-chunk / metadata filter / reranker / refresh corpus) → add the query to the probe set (and the case to the eval set) so it's guarded → gate/shadow/canary → roll out, confirming probe-set hit-rate@k recovered.
9. A provider silently shipped a new model snapshot (same name, changed behavior) — the fingerprint is a step change with no deploy of yours. Confirm by re-running your eval set ([Chapter 7](07-evaluation.md)) to size the regression; respond by re-pinning to a prior version if available, or revising prompt/retrieval to adapt, gated as usual.
10. Likely retrieval-quality decay: the corpus went stale relative to a shifted query mix, or the embedding model changed/was changed. Check first: run the probe set (does hit-rate@k confirm the drop and attribute it to retrieval?), then check for an embedding-model or corpus change, and sample the drifted live queries.
11. Re-embed the **entire** corpus with the new model and rebuild the index (query and chunk embeddings must share one model/space) — never partial. Prove it by running the probe-set hit-rate@k against baseline *before* serving, and gate/canary the reindex.
12. Non-deterministic agent paths can't be snapshot-and-replayed, and trace volume grew past where ad-hoc reading scales (~1k+ runs/day → tens of thousands of spans). You need a trace backend that searches/filters/samples spans, and you debug by per-step trace inspection, not log scrolling.
13. Quality fragments — divergent judges mean numbers aren't comparable, private eval sets mean one team's fix can silently regress another's cases ([Chapter 7](07-evaluation.md) vibes at org scale). Fix: one shared validated judge, one shared growing eval set, and versioned prompts in source control as the single source of truth.
14. Detectors on a schedule: nightly probe-set hit-rate@k, rolling query↔doc cosine, sampled faithfulness trend, refusal/thumbs-down rates, input-centroid drift. Triggers: any probe-set drop / >2σ cosine dip / thumbs-down cluster / refusal spike / provider-update notice fires a fix. A surfaced failure → open trace → triage into eval set → fix artifact → add the query to the probe set so it's permanently guarded → gate/canary.
15. Input-distribution drift = the *questions* change (measured by query-embedding centroid/cluster movement vs a baseline window) — a *leading* indicator that says "sample these into the eval set." Retrieval-quality decay = the *retriever returns worse chunks* (measured by query↔doc cosine >2σ drop and probe-set hit-rate@k). Different fixes: input shift → expand eval set / possibly grow the corpus; retrieval decay → re-chunk/re-embed/retune/rerank. Output-quality decay is the *lagging* indicator (harm already shipped); input shift is the earliest leading one.

**Deliverable:** a **monitoring + improvement-loop kit** for a prior build ([Chapter 4](04-rag.md) RAG bot or [Chapter 5](05-agents.md) agent): (a) a **≥50-item retrieval probe set** with a recorded **baseline hit-rate@k**; (b) a demonstrated **seeded regression** (garbled corpus *or* embedding-model change) showing the hit-rate@k drop and its attribution to retrieval-quality decay; (c) the **worked irrelevant-chunk path** ([Part D](#part-d--a-worked-path-handle-irrelevant-chunk-retrieval)) — a thumbs-down/probe failure traced to an off-topic chunk, **one** fix applied, and the probe-set hit-rate@k shown to **recover**, with that query added to the probe set; (d) at least one **online-judge faithfulness score attached to a trace** with a rolling trend; and (e) a one-paragraph **operating note**: which detectors you'd schedule, which signals trigger which artifact refresh (your [Part B](#part-b--re-pointing-retraining-for-build-on-top-retrain--refresh-the-artifacts-you-control) mapping), and how a surfaced failure becomes a permanent guard. *(Optional: the query-centroid drift number between two batches.)*

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
