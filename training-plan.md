# AI Training Plan

A top-down program for the team to learn modern AI / LLMs — start from the big picture,
then go deeper section by section. Daily updates go to Ayush via Slack DM (a one-liner is fine).

## Who this is for

We are training engineers to **build on top of LLMs** — to develop products and features that
use models via APIs — **not** to build or train LLMs themselves.

That sets the depth: internals (pretraining, the Transformer, reinforcement learning) are
covered only deep enough to make good engineering decisions. We never train a model. The real
depth of this program is in sections 2–9 (prompting, APIs, RAG, agents, adaptation, evaluation,
deployment). Every Foundation topic is taught through one lens: **"how does this change
something I'd build?"**

## Course spine

| # | Section | Day-block | Chapter |
|---|---------|-----------|---------|
| 1 | Foundations — How LLMs Actually Work | Day 1–2 | [01 — Foundations](chapters/01-day-1-2-foundations.md) |
| 2 | Using LLMs Well — Prompting & Interaction | Day 5–6 | [03 — Prompt Engineering](chapters/03-day-5-6-prompt-engineering.md) |
| 3 | Building with LLMs — APIs & Integration | Day 3–4 | [02 — APIs & Integration](chapters/02-day-3-4-apis-and-integration.md) |
| 4 | Grounding LLMs — RAG & Context | Day 7–8 | [04 — RAG](chapters/04-day-7-8-rag.md) |
| 5 | Agents & Tool Use | Day 9–10 | [05 — Agents](chapters/05-day-9-10-agents.md) |
| 6 | Customization — Fine-tuning & Adaptation | Day 11–12 | [06 — Customization](chapters/06-day-11-12-customization.md) |
| 7 | Evaluation, Safety & Reliability | Day 13–14 | [07 — Evaluation](chapters/07-day-13-14-evaluation.md) |
| 8 | Deployment & Production | Day 15–16 | [08 — Deployment](chapters/08-day-15-16-deployment.md) |
| 9 | Capstone Project | Day 17+ | _to be planned_ |

> The spine is ordered by topic; the day-blocks run in teaching order (APIs come early, on
> Day 3–4, so there's something concrete to prompt against). Read the chapters in day order:
> 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08.

## Chapters

1. [Day 1–2 — Foundations: how LLMs actually work](chapters/01-day-1-2-foundations.md)
2. [Day 3–4 — Talking to an LLM: from first call to production-ready](chapters/02-day-3-4-apis-and-integration.md)
3. [Day 5–6 — Prompt Engineering: reliable behavior on demand](chapters/03-day-5-6-prompt-engineering.md)
4. [Day 7–8 — Grounding LLMs: RAG & Context](chapters/04-day-7-8-rag.md)
5. [Day 9–10 — Agents & Tool Use: from single call to autonomous loop](chapters/05-day-9-10-agents.md)
6. [Day 11–12 — Customization: adapting a model without training one](chapters/06-day-11-12-customization.md)
7. [Day 13–14 — Evaluation, Safety & Reliability: shipping with confidence](chapters/07-day-13-14-evaluation.md)
8. [Day 15–16 — Deployment & Production: running an LLM feature for real](chapters/08-day-15-16-deployment.md)
9. Day 17+ — Capstone Project _(to be planned)_

## How each chapter is built

Every day-block follows the same shape so the team always knows where to look:

- **Goal** → **Why it matters** → a **setup-assumed** note
- **Suggested split** across the two days
- **Parts** (numbered concepts) — each concept ends with a *Build consequence:* line tying it
  back to something you'd actually build
- Side-by-side **Anthropic / OpenAI** Python where code applies
- **Resources** → **Hands-on tasks** → **Questions** (Check understanding / Apply it / Stretch)
  → **Answer key** → **Deliverable** → **Daily update (DM to Ayush)** → **Time estimate**

## Gaps & roadmap (planned additions)

The core spine (sections 1–8) is built. These additions close known gaps for engineers
**shipping** real LLM features. To avoid renumbering the live course, they land two ways:
**grafts** (new Parts inside existing chapters — purely additive) and **extension
chapters** (Days 17+ "Advanced & Production Track," alongside the Capstone).

> **v1 (current cohort):** append as below. **v2 (next cohort):** interleave at the
> *ideal slot* so security follows RAG/agents and governance precedes deployment.

### New chapters — Advanced & Production Track (Day 17+)

| Topic | Scope | v2 ideal slot |
|-------|-------|---------------|
| Beyond Text — Vision, Voice & Generation | vision-in (OCR, tables, charts, multi-image); audio-in (STT, diarization); audio-out (TTS); realtime voice agents (turn-taking, latency budget); image/video generation; provenance (C2PA); multimodal eval & cost | after ch03 |
| Frameworks, the Ecosystem & MCP | raw SDKs vs orchestration frameworks (LangChain, LlamaIndex, Vercel AI SDK, Pydantic AI); build-vs-framework decision; MCP (client/server/tools/resources); hands-on MCP server | after ch05 |
| Security, Privacy & Governance | what data may leave to a provider; PII redaction; residency; DPAs / zero-retention / training opt-out; GDPR, HIPAA, EU AI Act; LLM-app threat model; defense-in-depth map (which control lives where) | before ch08 |
| Memory & Personalization _(or a Part in ch05)_ | session vs cross-session state; what to store / what not to; summary buffers, memory store, RAG-over-history, structured profiles; personalization; user view/delete controls | after ch05 |
| LLM Application Architecture & System Design | workflows vs agents (when deterministic orchestration beats an autonomous loop); the orchestration patterns — prompt chaining, routing (LLM-as-classifier), parallelization (sectioning/voting), orchestrator-workers, evaluator-optimizer; the compound-AI-system framing (models + retrievers + tools + validators, not one big prompt); gateway / provider-abstraction layer (model-agnostic SDK boundary, fallback chains, model tiering/routing); reliability (retries, timeouts, circuit breakers, rate-limit/quota handling, graceful degradation); context engineering & the data plane (context window as a managed resource; ingestion/chunking/reindex as a pipeline service); caching tiers (prompt, semantic, response, embedding); guardrails as an input→model→output layer; streaming architecture (SSE/WebSocket, cancellation, partial rendering); observability (LLM spans, per-tenant cost attribution) & the feedback loop | between ch05 and ch08 |

### Grafts into existing chapters (additive Parts, no renumbering)

| Chapter | New Part | Hands-on |
|---------|----------|----------|
| 04 — RAG | Multi-tenant retrieval & isolation | tenant filter + a test proving user A can't retrieve user B's docs |
| 04 — RAG | Vector databases in depth — ANN index types & tradeoffs (HNSW, IVF/IVF-PQ, DiskANN; recall vs latency vs memory; the `ef_search`/`nprobe` knob); distance metrics (cosine vs dot vs L2) matched to the embedding model; store selection (dedicated — Pinecone/Weaviate/Qdrant/Milvus — vs `pgvector` vs OpenSearch/Mongo Atlas; managed vs self-hosted in Docker); metadata pre- vs post-filtering, namespaces/collections; quantization (scalar/product/binary) for memory & cost; hybrid search & fusion (BM25 + dense, RRF) in the DB; upserts, persistence, backups, and reindex/migration when the embedding model changes; scaling (sharding, replication, latency budget) | stand up `pgvector` in Docker, load the corpus, build an HNSW index; compare recall + p95 latency vs the brute-force store from Part C; add a metadata pre-filter and prove it scopes results |
| 05 — Agents | Tool & output safety | validate/allowlist tool args before execution; injection-via-tool-output test |
| 07 — Evaluation | Testing code vs evaluating the model | mock the LLM; deterministic suite separate from the eval set |
| 08 — Deployment | Concurrency & throughput; safe output rendering | parallel fan-out + throughput measure; sanitize a rendered output |
| 07 — Evaluation | The feedback loop: user signals → eval set | capture thumbs-up/down + traces, append them to the eval set |
| 08 — Deployment | Resilience: retries, timeouts, circuit breakers, provider fallback | wrap a call with retry+timeout+fallback model; force a 429/outage and prove graceful degradation |

### Rollout order

1. **The seven grafts** — small, additive, immediately useful to the cohort already in those chapters.
2. **Security, Privacy & Governance** — highest risk reduction.
3. **Frameworks & MCP** — most time-sensitive / 2026-relevant.
4. **LLM Application Architecture & System Design** — ties the whole spine together; highest leverage for engineers shipping real systems.
5. **Beyond Text** — biggest capability unlock, largest effort.
6. **Memory & Personalization.**
