# AI Training Plan

A top-down program for the team to learn modern AI / LLMs — start from the big picture,
then go deeper section by section. Each chapter ends with a short daily update (a one-liner is fine).

## Who this is for

We are training engineers to **build on top of LLMs** — to develop products and features that
use models via APIs — **not** to build or train LLMs themselves.

That sets the depth: internals (pretraining, the Transformer, reinforcement learning) are
covered only deep enough to make good engineering decisions. We never train a model. The real
depth of this program is in chapters 2–8 (APIs, prompting, RAG, agents, adaptation, evaluation,
deployment). Every Foundation topic is taught through one lens: **"how does this change
something I'd build?"**

## Course spine

Chapters are numbered in reading order — work straight through 1 → 8, then the Capstone.
APIs come early (Chapter 2) so there's something concrete to prompt against in Chapter 3.

| # | Chapter | Focus |
|---|---------|-------|
| 1 | [Foundations — How LLMs Actually Work](chapters/01-foundations.md) | the mental model: tokens, the Transformer, context, sampling |
| 2 | [Talking to an LLM — APIs & Integration](chapters/02-apis-and-integration.md) | first call → structured output, tool calling, error handling |
| 3 | [Prompt Engineering](chapters/03-prompt-engineering.md) | reliable behavior on demand |
| 4 | [Grounding LLMs — RAG & Context](chapters/04-rag.md) | answer from your data, with citations |
| 5 | [Agents & Tool Use](chapters/05-agents.md) | the model in a loop, choosing and running tools |
| 6 | [Customization — Fine-tuning & Adaptation](chapters/06-customization.md) | the adaptation ladder; pick the right rung |
| 7 | [Evaluation, Safety & Reliability](chapters/07-evaluation.md) | shipping with confidence, not vibes |
| 8 | [Deployment & Production](chapters/08-deployment.md) | run an LLM feature for real |
| 9 | Capstone Project | _to be planned_ |

## Chapters

1. [Foundations: how LLMs actually work](chapters/01-foundations.md)
2. [Talking to an LLM: from first call to production-ready](chapters/02-apis-and-integration.md)
3. [Prompt Engineering: reliable behavior on demand](chapters/03-prompt-engineering.md)
4. [Grounding LLMs: RAG & Context](chapters/04-rag.md)
5. [Agents & Tool Use: from single call to autonomous loop](chapters/05-agents.md)
6. [Customization: adapting a model without training one](chapters/06-customization.md)
7. [Evaluation, Safety & Reliability: shipping with confidence](chapters/07-evaluation.md)
8. [Deployment & Production: running an LLM feature for real](chapters/08-deployment.md)
9. Capstone Project _(to be planned)_

### Advanced & Production Track

Extension chapters that close known gaps for engineers shipping real LLM features,
drafted alongside the Capstone (the rest of the track is backlogged in _Gaps & roadmap_ below):

1. [Cloud Managed-GenAI Platforms](chapters/adv-cloud-managed-genai-platforms.md)
2. [Monitoring, Drift & the Continuous-Improvement Loop](chapters/adv-monitoring-drift-loop.md)

## How each chapter is built

Every chapter follows the same shape so the team always knows where to look:

- **Goal** → **Why it matters** → a **setup-assumed** note
- **Suggested split** across two working sessions
- **Parts** (numbered concepts) — each concept ends with a *Build consequence:* line tying it
  back to something you'd actually build
- Side-by-side **Anthropic / OpenAI** Python where code applies
- **Resources** → **Hands-on tasks** → **Questions** (Check understanding / Apply it / Stretch)
  → **Answer key** → **Deliverable** → **Daily update** → **Time estimate**

## Gaps & roadmap (planned additions)

The core spine (chapters 1–8) is built. These additions close known gaps for engineers
**shipping** real LLM features. To avoid renumbering the live course, they land two ways:
**grafts** (new Parts inside existing chapters — purely additive) and **extension
chapters** (the "Advanced & Production Track," alongside the Capstone).

> **v1 (current cohort):** append as below. **v2 (next cohort):** interleave at the
> *ideal slot* so security follows RAG/agents and governance precedes deployment.

> **Status:** the first wave is now **drafted** (marked ✅) — two extension chapters and ten
> grafts derived from a review of ~3,000 real Gen-AI interview questions. Frameworks/libraries
> (LangChain, LlamaIndex, …) are intentionally deferred. The rest (marked ⬜) is still planned.

### New chapters — Advanced & Production Track

| Topic | Scope | v2 ideal slot |
|-------|-------|---------------|
| Beyond Text — Vision, Voice & Generation | vision-in (OCR, tables, charts, multi-image); audio-in (STT, diarization); audio-out (TTS); realtime voice agents (turn-taking, latency budget); image/video generation; provenance (C2PA); multimodal eval & cost | after ch03 |
| Frameworks, the Ecosystem & MCP | raw SDKs vs orchestration frameworks (LangChain, LlamaIndex, Vercel AI SDK, Pydantic AI); build-vs-framework decision; MCP (client/server/tools/resources); hands-on MCP server | after ch05 |
| Security, Privacy & Governance | the **OWASP LLM Top-10 (2025)** as shared vocabulary + the **defense-in-depth map** — controls at four stations (input → model → output → action) chosen by failure mode. Data governance / leak-to-provider: pre-call **PII/PHI redaction** (Presidio + custom clinical recognizers), self-host vs **ZDR / DPA / BAA / training-opt-out / residency**, per-provider retention defaults. **PHI & HIPAA** (BAA; Safe-Harbor vs Expert-Determination de-identification). **Prompt injection in depth** — direct vs **indirect** (poisoned retrieved doc / tool output / web page / image); spotlighting, instruction hierarchy, injection classifiers (Azure Prompt Shields, Llama Prompt Guard), and the **dual-LLM / CaMeL** architectural defense. **Output guardrails** (leak/PII scan, content-safety, grounding check, de-anonymize). **Guardrail engineering / "which guardrail"** — build-vs-buy menu (Presidio, Llama Guard, NeMo Guardrails, Guardrails AI, Azure AI Content Safety, AWS Bedrock Guardrails) + risk→station rule. **API & cloud security** (API-key vs OAuth2/OIDC, per-user-token isolation, **token-based rate limiting** for cost/DoS, audit trails). **Regulatory** (GDPR; HIPAA; **EU AI Act** — enforcement + high-risk obligations from 2026-08-02, Article 50 transparency, fines to €35M/7%). Governance as engineering (versioning, eval gates, audit logs, incident response, a written threat model). | before ch08 |
| Memory & Personalization _(or a Part in ch05)_ | session vs cross-session state; what to store / what not to; summary buffers, memory store, RAG-over-history, structured profiles; personalization; user view/delete controls | after ch05 |
| LLM Application Architecture & System Design | workflows vs agents (when deterministic orchestration beats an autonomous loop); the orchestration patterns — prompt chaining, routing (LLM-as-classifier), parallelization (sectioning/voting), orchestrator-workers, evaluator-optimizer; the compound-AI-system framing (models + retrievers + tools + validators, not one big prompt); gateway / provider-abstraction layer (model-agnostic SDK boundary, fallback chains, model tiering/routing); reliability (retries, timeouts, circuit breakers, rate-limit/quota handling, graceful degradation); context engineering & the data plane (context window as a managed resource; ingestion/chunking/reindex as a pipeline service); caching tiers (prompt, semantic, response, embedding); guardrails as an input→model→output layer; streaming architecture (SSE/WebSocket, cancellation, partial rendering); observability (LLM spans, per-tenant cost attribution) & the feedback loop | between ch05 and ch08 |
| ✅ [Cloud Managed-GenAI Platforms](chapters/adv-cloud-managed-genai-platforms.md) | the hyperscaler GenAI menu for engineers who *consume* (not train) models: consume-via-API (Bedrock, Azure OpenAI / Microsoft Foundry, Vertex) vs host-your-own (SageMaker, Azure ML, Vertex endpoints); reading the catalogs (incl. 2026 rebrands + the Assistants→Foundry Agent Service retirement); **serverless inference economics** (scale-to-zero vs always-on vs provisioned-concurrency — the "does it cost when idle?" answer); GPU instance sizing; managed model registry / MLflow; multi-region HA as config; the managed-vs-self-host decision | after ch08 |
| ✅ [Monitoring, Drift & the Continuous-Improvement Loop](chapters/adv-monitoring-drift-loop.md) | the running-in-prod loop for engineers who don't own the weights: **drift re-pointed** to three app-level decays (input-distribution shift, retrieval-quality decay via probe sets, output-quality decay); **"retraining" re-pointed** to refreshing corpus/prompts/few-shots/model-pins (event-driven); the full loop — trace → online eval → triage → fix the right artifact → gate → shadow/canary → roll out/back; production RAG ops & agentic-prod challenges | between Architecture & Capstone |

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
| ✅ 01 — Foundations | Model families: encoder vs decoder (and where embeddings come from) | contextual-embedding demo ("river bank" vs "savings bank"); map the chat / embedding / reranker models onto decoder vs encoder |
| ✅ 03 — Prompt Engineering | Redact PII before the call (don't send what you don't need) | run user text through a redactor (Presidio) before sending; restore the placeholders in the output |
| ✅ 04 — RAG | Document ingestion & extraction (the front half of the pipeline) | Docling-in-Docker on a messy multi-page PDF with tables + a scanned page; OCR off→on; feed into the brute-force store; 5-field extraction-accuracy compare |
| ✅ 04 — RAG | Choosing an embedding model + cross-encoder reranking | embedding bake-off (hit-rate@4); add a cross-encoder reranker (rank promotion); Matryoshka half-dim truncation tradeoff |
| ✅ 04 — RAG | Retrieved text is untrusted data (indirect prompt injection) | poison a corpus doc; show the naive RAG prompt obeys it; defend with spotlighting + an injection check |
| ✅ 04 — RAG | Managed RAG pipelines (cloud-native) | stand up the corpus as Bedrock KB / Vertex RAG Engine / Azure AI Search; compare answer quality + setup effort vs the hand-built pipeline |
| ✅ 05 — Agents | Managed agent runtimes (cloud-native agents) | re-implement the hand-built agent as a Bedrock Agent / Vertex Agent Engine; compare lines-of-code / latency / flexibility |
| ✅ 07 — Evaluation | Online eval & evaluating without a gold answer | sample 20% of prod, run the reference-free faithfulness judge → live hallucination rate; thumbs-down → trace → fix the right artifact |
| ✅ 07 — Evaluation | Hallucination: a named taxonomy, detection & the BLEU/ROUGE reframe | span-level faithfulness check; classify intrinsic vs extrinsic; SelfCheckGPT self-consistency on a no-context question |
| ✅ 08 — Deployment | Tracing with spans (OTel GenAI) + shadow-mode rollout | parent/child spans on the Day-15 endpoint; shadow a 2nd prompt version, score it, make a go/no-go call |

### Rollout order

Legend: ✅ drafted (in the repo) · ⬜ planned.

1. ✅ **The ten new grafts** — encoder/decoder (ch01), PII redaction (ch03), document ingestion, embedding-model choice + reranking, indirect injection, managed RAG (ch04), managed agent runtimes (ch05), online eval + hallucination (ch07), span tracing + shadow rollout (ch08). Small, additive, immediately useful to the cohort already in those chapters.
2. ✅ **Cloud Managed-GenAI Platforms** — the biggest standalone gap; the cloud-native sequel to ch08.
3. ✅ **Monitoring, Drift & the Continuous-Improvement Loop** — the running-in-prod loop you operate after architecting and deploying.
4. ⬜ **The seven earlier grafts** (multi-tenant retrieval, tool/output safety, testing-vs-eval, concurrency/safe-render, vector-DB depth, feedback loop, resilience) — still to write.
5. ⬜ **Security, Privacy & Governance** — highest risk reduction; the *injection* + *PII-redaction* reflexes are already grafted into ch03/ch04, keep the full chapter before ch08.
6. ⬜ **Frameworks & MCP** — most time-sensitive / 2026-relevant. (Frameworks/libraries intentionally deferred for now.)
7. ⬜ **LLM Application Architecture & System Design** — ties the whole spine together; highest leverage for engineers shipping real systems.
8. ⬜ **Beyond Text** — biggest capability unlock, largest effort.
9. ⬜ **Memory & Personalization.**
