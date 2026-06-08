# Authoring Plan — the remaining curriculum

How we write the rest of the course (6 grafts + 5 extension chapters + the Capstone) so that
**every page is understandable to a learner new to LLMs.** This plan was produced by reviewing
the existing chapters for what makes them teachable, then planning each remaining piece against
that same bar.

Audience reminder: we train engineers to **build on top of LLMs via APIs** — they never train a
model. Depth is calibrated to "how does this change something I'd build?"

---

## The learner-understandability contract (every piece must pass this)

This is the bar derived from chapters 1, 4, 7. Treat it as a checklist before any chapter ships.

1. **One bold claim per concept.** Each numbered concept opens with a single declarative
   sentence (the takeaway), then explains, then closes with a `*Build consequence:*` line naming
   a concrete design/debug decision the API-consumer owns. *If a concept can't earn a build
   consequence, cut it — it's too academic for this audience.*
2. **A vivid, correct analogy before any mechanism.** One image per hard idea (temperature = a
   loaded die; context window = working memory; guardrails = a sandwich; reranker = a smarter
   second pass). Lead with the analogy, then the code.
3. **Derive, don't assert.** Build each idea from the crack in the previous one (exact-match
   breaks → normalize → classification → accuracy is a trap → confusion matrix → P/R → F1). The
   learner should see *why* a tool exists so they can choose it under novel conditions.
4. **Worked numbers, hand-checkable, chosen to expose the trap** (the 0.94 accuracy that
   shouldn't ship; the 1.8s→0.6s fan-out; the voice latency budget summing past 1s).
5. **Decision ladders + tables.** Turn every menu into a defaults-first ordered procedure ("start
   here → escalate when ___"). Always name a sane default and the trigger to step up.
6. **Name the anti-pattern.** Pre-empt the predictable beginner mistake explicitly ("don't crank
   temperature to fix wrong answers"; "don't start with a vector DB"; "agent-washing").
7. **`> Setup assumed` notes** hoist boilerplate so code stays short and the concept stays in focus.
8. **Side-by-side Anthropic / OpenAI + "from scratch" once.** Provider-neutral, and de-black-box
   the mechanism (cosine in one line of NumPy; pgvector via raw SQL) before reaching for a library.
9. **Stable interfaces across hands-on tasks.** Everything hides behind one signature
   (`retrieve(question)->chunks`) so later tasks build on earlier artifacts as drop-in swaps.
10. **"Deepens X, does NOT re-cover Y."** Every graft/extension opens by stating which earlier
    concepts it builds on and what it deliberately won't repeat — clickable cross-refs with scope.
11. **Three-tier questions + gated answer key** (Check / Apply / Stretch; "peek only after
    attempting") so a solo learner can self-assess recall, application, and judgment.
12. **Expand jargon at first use** (IDOR, fail-closed, circuit breaker, half-open, UGC, WER, C2PA).
13. **Fixed chapter tail** so the reader always knows where to look:
    Goal → Why it matters → Setup assumed → Suggested split → Parts → Resources → Hands-on →
    Questions (3 tiers) → Answer key → Deliverable → Daily update → Time.
14. **House voice:** calm senior-to-peer, no manufactured excitement; confidence from precision.

---

## What remains (scope at a glance)

| # | Piece | Type | Parts | Est. words | Effort |
|---|-------|------|:-----:|:----------:|:------:|
| 1 | Six remaining grafts | grafts | 6 | 6–8k | large |
| 2 | Security, Privacy & Governance | chapter | 9 | 8.5–9.5k | large |
| 3 | Frameworks, the Ecosystem & MCP | chapter | 5 | 4.5–5.5k | medium |
| 4 | LLM Application Architecture & System Design | chapter | 6 | 5.5–6.5k | large |
| 5 | Beyond Text — Vision, Voice & Generation | chapter | 7 | 7.5–9k | large |
| 6 | Memory & Personalization | chapter | 9 | 5.5–7k | small* |
| 7 | Capstone Project (Chapter 9) | capstone | 6 | 3.5–4.5k | large |

\* small *effort* (deepens existing concepts) despite 9 short Parts. Total new content ≈ 41–50k words.

---

## Authoring sequence (waves, by dependency + value)

- **Wave 1 — Six grafts.** Cheapest, additive, immediately useful to the cohort already in those
  chapters. Order: the two ch07 grafts together (share the eval-set artifact), then the two ch08
  grafts together (share the production endpoint), then the standalone ch04 and ch05 grafts.
- **Wave 2 — Security, Privacy & Governance.** Highest risk-reduction. Its PII-redaction (ch03)
  and indirect-injection (ch04) reflexes are already grafted, so it's the "full treatment."
- **Wave 3 — Frameworks & MCP.** Most time-sensitive / 2026-relevant. Ship MCP first; frameworks
  fenced as optional/deferred for this cohort.
- **Wave 4 — LLM Application Architecture.** Highest leverage; ties the spine together.
- **Wave 5 — Beyond Text.** Biggest capability unlock, largest effort.
- **Wave 6 — Memory & Personalization.** Small effort; natural home for the privacy reflex.
- **Wave 7 — Capstone.** Last; reuses every chapter's artifact. Write after the grafts land if
  possible (depends only on the built spine 1–8, which is done).

---

## 1 — Six remaining grafts

Append to host chapters, continuing global concept numbering, each opening with the
"deepens / does-not-re-cover" ritual. ch04 → new Part L; ch05 → new Part H; ch07 → Parts I, J;
ch08 → Parts H, I.

| # | Host | Part | Core idea | Understandability hook | Hands-on |
|---|------|------|-----------|------------------------|----------|
| 1 | ch04 | Multi-tenant retrieval & isolation | "Relevant" and "allowed" are different axes; isolation is a hard **pre-filter** before ranking, derived from the auth context, never the request body | Keycard-on-a-drawer; worked 1000-chunk example showing post-filter both *starves* top-k and *leaks* the other tenant's chunk | `retrieve(q, tenant_id)` with tenant_id from auth; a test proving user A can't read B's docs + a missing-tenant fails-closed test |
| 2 | ch05 | Tool & output safety | A tool call is attacker-reachable on both sides: validate/allowlist **args** before execution; label tool **output** as untrusted (indirect injection via a tool) | Customs checkpoint (X-ray luggage in, stamp mail "unverified" out); derive from `tool_impls[name](**input)` being an injection sink | Arg-allowlist on one risky tool (rejects with a recoverable error); injection-via-tool-output test run raw vs labelled |
| 3 | ch07 | Testing code vs evaluating the model | Two correctness kinds, two suites: mock the LLM to test deterministic **code**; use the eval set to measure **model** behavior | Test the plumbing vs taste the water; show one flaky `assert == 'Paris'` test secretly doing both jobs | Deterministic mocked unit test (no API key) kept separate from the live-model eval set |
| 4 | ch07 | The feedback loop: signals → eval cases | Build concept-27's flywheel: capture trace + user signal, triage negatives, promote **human-verified** cases (never the model's wrong output) | Production = a bug tracker that files itself, but a 👎 is an unverified lead you triage; Goodhart warning on optimizing the signal | Capture 👍/👎 + trace, walk one 👎 to the broken component, promote a human-corrected (input, expected) case |
| 5 | ch08 | Concurrency & throughput; safe rendering | Throughput = **bounded** concurrency (semaphore sized to TPM), not "fire everything"; model output is untrusted UGC at the render boundary | Kitchen burners + gas line (429 storm); worked 1.8s→0.6s fan-out then unbounded→429; `<img onerror>` rendered raw vs escaped | Parallel fan-out + throughput measure behind a semaphore; sanitize an injected `<script>`/`javascript:` output |
| 6 | ch08 | Resilience: retries, timeouts, breakers, fallback | The ordered stack — timeout → backoff+jitter retry → **circuit breaker** (the missing piece) → fallback/graceful degradation | Building fire safety (detector/try-door/auto-shutoff/marked-exit); derive ordering by what each layer fixes that the prior can't | Wrap a call in the full stack; force 429/hang/hard-down and prove transient-retry, breaker-trips-fast, degrades-gracefully |

**Cross-cutting confusions to head off:** isolation-is-a-ranking-problem; the-model-can-be-*instructed*-into-safety (it's not a security boundary); a-mocked-suite-measures-quality; enshrining-the-bug as expected; fire-everything-in-parallel + retries-alone-equal-resilience; output-is-trusted-because-it's-our-model.

---

## 2 — Security, Privacy & Governance
*Chapter; slot before ch08. Spine artifact = a four-station control map the chapter fills in.*

**Goal/deliverable:** a one-page **threat model** + a filled-in **four-station control map** for a
feature the learner already built.

| Part | Idea | Understandability hook |
|------|------|------------------------|
| A — Vocabulary + the map | OWASP LLM Top-10 (2025) as a shared dictionary + refine ch07's input→model→output sandwich into **four stations** (add **action**) | Customs checkpoints; motivate the 4th station from the crack — an injection that only *talks* is harmless, one that *acts* is the breach |
| B — Data governance | Two orthogonal levers that **stack**: send less (redact pre-call) + contract for less (ZDR/DPA/residency) | Mailing a postcard through a mail room; extend ch03 redaction with reversible tokenization + a custom recognizer |
| C — PHI & HIPAA | Redaction becomes a legal de-identification standard; a BAA is mandatory or it's a breach by definition | Safe Harbor = a codeable 18-item checklist (default) vs Expert Determination = a statistician signs off; "every copy" trace |
| D — Injection in depth | Climb the ladder to the rung ch3/4 deferred: **architectural** defenses (dual-LLM / CaMeL) | Security-cleared officer + untrusted clerk; worked email-agent attack the classifier misses but architecture stops |
| E — Output guardrails | The output station: ordered leak-scan → moderate → ground-check → de-anonymize; where fail-safe-vs-fail-open is decided | Airport exit security; show what breaks if you reorder de-anonymize before the scan |
| F — Which guardrail? | One rule resolves the market: **risk → station → tool**; compose 2–3, don't adopt one framework | Decision table over Presidio/Llama Guard/NeMo/Guardrails AI/Azure/Bedrock; buy only for a station with a named failure |
| G — API & cloud security | API-key vs OAuth2/OIDC, per-user-token isolation, **denial-of-wallet** rate limiting, audit trails | Cost-based abuse as the LLM-specific reframe of DoS |
| H — Regulation you can act on | GDPR (right-to-erasure across every sink), HIPAA (recap), EU AI Act (risk tiers, Art. 50, 2026-08-02, €35M/7%) | Walk *your own* feature down the risk-tier ladder; the medical-hiring bot triggering all three regimes at once |
| I — Governance as engineering | Governance = versioning + eval gates + audit logs + incident response, re-aimed; the threat model is the keystone | The chapter ends where it began — the Part-A four-station diagram, now filled in |

**Key confusions:** redaction-vs-contract are alternatives (they stack); a classifier "solves" injection; de-identifying only the input (raw PII still in store/logs/cache); the EU AI Act is "for model trainers"; the action station is "just a fancier sandwich"; Safe Harbor vs Expert Determination.

---

## 3 — Frameworks, the Ecosystem & MCP
*Chapter; slot after ch05. File: `chapters/adv-frameworks-ecosystem-mcp.md`. **MCP is the core deliverable; frameworks fenced as optional/deferred.***

**Goal:** (1) decide raw-SDK vs framework and defend it; (2) build + run an MCP server exposing a
tool and a resource, wired to a real host AND your own ch05 loop.

| Part | Idea | Understandability hook |
|------|------|------------------------|
| A — Raw SDKs vs frameworks *(optional/deferred)* | A framework just pre-packages the loops you already hand-wrote; decide build-vs-buy on an abstraction you understand from the inside | "You already wrote a framework" reframe; defaults-first ladder; year-stamp the churn |
| B — The integration problem MCP solves | N×M bespoke integrations → N+M via one protocol | USB-C plug shape / HTTP; derive 3×4=12 vs 3+4=7 arithmetic; "don't confuse MCP with tool calling — they stack" |
| C — MCP anatomy | 3 roles (host/client/server) + 3 primitives (tools/resources/prompts) by **who controls each** | Restaurant: diner/waiter/kitchen; show JSON-RPC **once** read-only ("you never write this, the SDK does") |
| D — Build an MCP server *(center of gravity)* | Mostly declaring + decorating functions; SDK handles the protocol | Incremental: toy `add()` → wrap ch04 `retrieve()` as `search_docs` → add a resource → recoverable error |
| E — Wire it up + security reality | A server is useful once a host consumes it; consuming *others'* servers inherits a supply-chain/permissions problem | Tool descriptions become untrusted input; adoption ladder official→vetted→self-built, sandbox the rest |

**Key confusions:** MCP = another framework / a LangChain competitor; MCP replaces tool calling; tools-vs-resources-vs-prompts lumped together; host/client/server blur; must hand-write JSON-RPC; "it's a standard so it's safe to connect"; this cohort must learn LangChain deeply now.

---

## 4 — LLM Application Architecture & System Design
*Chapter; slot between ch05 and ch08. File: `chapters/adv-architecture-system-design.md`. Spine artifact = one "support assistant" diagram that accretes across all 6 Parts.*

**Goal:** take a vague request and **draw its architecture** — deterministic vs autonomous,
the right orchestration pattern per step, the cross-cutting layers as named components, each box
justified by the failure it prevents.

| Part | Idea | Understandability hook |
|------|------|------------------------|
| A — Compound AI systems | The system (components behind clean interfaces), not the prompt, is the unit | One-big-prompt = a 2000-line function with no subroutines; redraw the support bot as 4 testable boxes |
| B — Workflows vs agents at system level | Most of a system is deterministic workflow; an agent is one island you justify | "Push work down the ladder" generalized; name **agent-washing** as the anti-pattern |
| C — The five orchestration patterns | Chaining, routing, parallelization (sectioning/voting), orchestrator-workers, evaluator-optimizer | Each: claim → diagram → crack-in-previous → 6–10 line sketch; symptom→pattern decision table |
| D — The gateway / provider boundary | One model-agnostic seam makes "which provider" a config change; unlocks fallback + tiering | Universal power adapter; build a ~15-line `llm_call()` normalizing Anthropic/OpenAI |
| E — Cross-cutting layers | Caching tiers + guardrail sandwich + the context/data plane as named bands wrapping the diagram | Cache **safety ladder** (semantic cache's "wrong hit = confidently wrong"); split read path from reindex/maintenance path |
| F — Streaming + observability | Streaming as a transport decision (SSE vs WebSocket); spans across the whole system + per-tenant cost | "Does the client send mid-stream?"; cancellation stops the spend; generalize ch08 spans to a span *tree* |

**Key confusions:** "architecture" = more code/a framework; agent-as-default; orchestrator-workers ≡ multi-agent / routing ≡ agent; semantic cache is "free like prompt cache"; ingestion runs per request; expecting deep retries/drift here (placed here, **operated** in ch08 + Resilience graft + Monitoring chapter).

---

## 5 — Beyond Text — Vision, Voice & Generation
*Chapter; slot after ch03 (its Part G multimodal paragraph is the seed). Largest chapter. Spine = one pipeline: **encode → shared tokens → decode**.*

**Goal:** vision-in (OCR/tables/charts/multi-image), audio-in (STT + diarization), audio-out (TTS),
the realtime voice **latency budget**, image/video generation, **C2PA** provenance, and multimodal
eval + cost. Verb stays CALL/CONSUME — never train.

| Part | Idea | Understandability hook |
|------|------|------------------------|
| A — One mental model | Every modality = encoder→tokens→decoder; native model vs specialist pipeline | Universal translator booth; defaults-first ladder (native first, drop to a specialist for a specific knob) |
| B — Vision-in | Image + schema-enforced output replaces brittle OCR — but it can silently mis-read, so verify | "OCR keeps characters, loses structure; LLM keeps structure, can hallucinate a character"; invoice line-items must sum to the total |
| C — Audio-in (STT + diarization) | Transcription and **diarization are different jobs** that fail independently | Stenographer vs gallery-watcher; worked transcript with perfect words but swapped speakers; WER defined inline |
| D — Audio-out (TTS) | One call; the new decision is streaming vs full-file; synthetic voice carries consent/provenance | Decode-side mirror of Part C; "is a human waiting for the first syllable?" |
| E — Realtime voice agents *(centerpiece)* | The round trip must fit a ~1s turn-taking budget; the naive chain blows it | Phone-silence rule; sum the budget in hand-checkable ms; ladder: stream/overlap → VAD/barge-in → realtime speech-to-speech API |
| F — Image/video generation + provenance | Generation is an API call with its own knobs + economics (per image/job); output must carry **C2PA** | Seed = the ch1 loaded die; C2PA = a tamper-evident nutrition label; "don't confuse C2PA manifest vs invisible watermark" |
| G — Multimodal eval & cost | Can't exact-match an image/transcript; new billing units | Derive from the crack in ch07's exact-match ladder; per-modality metric recipe; worked mixed-modality monthly cost |

**Key confusions:** "multimodal = one magic model"; transcription ≡ diarization; voice agent as a synchronous request/response; a vision model is as reliable as OCR; costing multimodal in tokens by habit; exact-matching image/voice output; C2PA ≡ watermark / optional polish.

---

## 6 — Memory & Personalization
*Chapter (recommended over a ch05 graft — cross-session memory applies to chat/RAG/agent alike). Slot after ch05. File: `chapters/adv-memory-personalization.md`. Spine = one user's fact lived through its whole lifecycle.*

**Goal:** give an app a memory that survives sessions **without retraining** — decide what to
persist/drop, implement four patterns behind one interface, assemble a budgeted per-turn context,
personalize at prompt-time, and ship view/export/edit/**delete** controls.

| Part | Idea | Understandability hook |
|------|------|------------------------|
| A — Standalone chapter, not a ch05 Part | Resolve the scope question; state the deepens/does-not-re-cover contract | "What the worker remembers during one task" vs "what the company remembers across visits" |
| B — Session vs cross-session | The model has neither for free; both are notes YOU keep and re-inject | Barista: oat-milk-this-order vs the regular's "usual" on a card; two clocks |
| C — What to store, louder what NOT to | Hoarding fails on cost, quality, privacy at once | Derive from 3 concrete failures; "memory as a junk drawer"; a wrong stored fact = a *persistent* hallucination |
| D — Four patterns behind one interface | Summary buffer / raw log / RAG-over-history / structured profile, each fixing the last's crack | "previously on…" recap / security tape / search the tape / form fields; reuse ch04 `retrieve()` with a user_id pre-filter |
| E — Assemble the turn (budget, not dump) | Curate a fixed-size working set by priority under a token budget | Packing a carry-on; worked 4k-budget allocation; ch4 "lost in the middle" from your own store |
| F — Maintenance: update/decay/conflict | Memory rots; supersede-by-key + recency weighting + `written_at` on everything | Beginner→expert scenario where stale memory patronizes the user; wiki-with-history not sticky notes |
| G — Personalization | Memory is input; personalization is output — and it's **prompt-time, not training-time** | Same query with empty vs populated profile; "don't confuse personalization (deletable) with fine-tuning (baked into weights)" |
| H — A memory store is a personal-data store | Relabel the shiny store "a database of personal facts, indefinitely" — that's the lesson | Map each capability to a governance duty; "encrypt vs let-the-user-delete are different obligations" |
| I — User controls (view/export/edit/delete) | The climax is **delete**: it must fan out across all four stores + summary + vector re-index | "Forget I work at Acme" lives in profile + log + embedding + summary; fail toward over-deletion |

**Key confusions:** the model "remembers" once you flip a feature; ch05 within-task memory ≡ cross-session; more context = smarter; personalize = fine-tune per user; memory self-corrects; "delete" = one profile row.

---

## 7 — Capstone Project (Chapter 9)
*Final chapter, after ch08. Reuses every chapter's artifact. **Adds zero new mechanisms** — only choices to make and defend, and the integration seams.*

**Goal/deliverable:** one shipped, defensible feature = a running service + an eval report + a
one-page design-decision log. Graded on **integration + defense, not novelty.**

**Part A — pick a brief (defaults-first):** Brief 1 *Docs Copilot* (grounded Q&A, RAG-heavy — the
safe default), Brief 2 *Research Agent* (multi-step tool use, ch05-heavy), Brief 3 *Structured
Extractor + Reviewer* (structured-output + eval-heavy). Each names a free corpus, a happy path, and
a "must refuse" line. Output: `SCOPE.md`.

**Milestones (B–F):**
- **B — Ground it:** point ch04 `retrieve()` at the real corpus; make the **refusal contract** a
  checkable behavior (cites in-corpus, refuses out-of-corpus). *Reuse, don't rebuild.*
- **C — Choose control flow:** fill a decision table that outputs the right rung; build the
  **least-autonomous** flow that works, bounded + traced. "I need rung X because X-1 cannot ___."
- **D — Prove it with a number:** ≥15-case eval set, metric matched to output shape, a baseline,
  then one deliberate change that **moves the number**. Names the accuracy-on-open-ended trap.
- **E — Production shape:** client→backend→provider, key server-side, full request logging, one
  forced-failure demo proving retry/timeout/fallback.
- **F — Defend it:** a one-page **design-decision log** (choice + build consequence + rejected
  alternative, ≥5 decisions), self-score against a weighted **rubric**, tick a binary
  **Definition of Done**.

**Rubric dimensions:** Grounding & refusal · Right-sized control flow · Evaluation rigor ·
Production shape · Defense quality.

**Key confusions:** "build something new and impressive" (it's integrate + defend); reaching for an
agent under capstone pressure; reporting one "accuracy" / a demo screenshot; "it runs in my
notebook" = deployed; skipping the written defense.

---

_Plan generated 2026-06-05 from a multi-agent pass: one agent extracted the understandability
rubric from the existing chapters; seven planned each remaining piece against it. Update the
`Gaps & roadmap` rollout markers in [training-plan.md](training-plan.md) as each ships._
