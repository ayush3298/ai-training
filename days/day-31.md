# Day 31 — Cloud Platforms, part 1 — consume-vs-host, catalogs & serverless economics

> [← Day 30](day-30.md) · [All days](README.md) · [Day 32 →](day-32.md)

**Module:** Cloud Managed-GenAI Platforms  ·  **Time:** ~2.5–3 hrs

## About this module

### Advanced Track — Cloud Managed-GenAI Platforms: choosing, sizing & costing a service

**Goal:** Be able to walk into any of the three big clouds (AWS, Azure, GCP), read its confusing GenAI catalog, and answer the two questions that actually matter for a build: *which* service do I use, and *what will it cost at my traffic?* By the end you can tell a "consume a provider model via API, zero infra" service apart from a "host your own model on instances you size" service on sight, explain serverless-vs-always-on-vs-provisioned inference economics (including the precise answer to "does it still cost money when idle?"), size a GPU job, and write a defensible managed-API-vs-self-host decision with a real monthly number attached.

**Why this matters:** [Chapter 8](08-deployment.md) taught you to *operate* a service that calls a model — but it assumed you'd already picked the provider API and were calling it directly. In a real org you rarely get that clean choice. You're told "we're an AWS shop" or "everything must run in our Azure tenant," and now you have to map your [Chapter 4](04-rag.md) RAG bot or [Chapter 5](05-agents.md) agent onto **Bedrock or SageMaker**, **Azure OpenAI or Microsoft Foundry**, **Vertex or a Vertex endpoint** — names that overlap, rebrand, and bill in completely different shapes. Pick wrong and you either pay for idle GPUs you didn't need or hit a wall self-hosting a model you should have just called. The skill here is not "train a model" (you don't) — it's **catalog literacy + cost arithmetic**: which box on the menu is which, and what the meter does when no one is using it.

> **Setup assumed:** [Chapter 8](08-deployment.md) (Deployment) done — you have a deployable build and you understand client → backend → provider, streaming, retries, per-request cost logging. You have an account in at least one cloud (AWS / Azure / GCP) with billing enabled, the relevant CLI/SDK installed (`boto3`, `azure-ai-*`/`openai`, `google-cloud-aiplatform`), and *model access requested* (Bedrock model access, Azure OpenAI deployment, or a Vertex/Gemini project) — access approval can take hours to a day, so request it first. The hands-on can run almost entirely **local-first in Docker** for the self-hosting half; the managed-API calls hit real endpoints (cheap, pennies). This is a cloud-economics chapter, not an MLOps course — depth is "enough to choose and size," not "enough to run a GPU fleet."

> **Two hard 2026 dates to put on the calendar now (callout):**
> - **Azure OpenAI Assistants API retires 2026-08-26.** It is being replaced by the **Microsoft Foundry Agent Service** (Threads → Conversations, Runs → Responses, Assistants → Agent Versions). This deadline is *not* opt-in or reversible — if you build on Assistants today you are building on something with a known end date. Build new agent work on the Foundry Agent Service / Responses API.
> - **Amazon Bedrock AgentCore went GA on 2025-10-13** (after a mid-2025 preview). It is now a real, billable, production option for hosting/operating agents — not a preview you should hedge against.
>
> *Names are in flux in 2026.* Google rebranded **Vertex AI → "Gemini Enterprise Agent Platform"** (announced 2026-04-22; Model Garden, Endpoints, Model Registry, AutoML all folded under it). Microsoft rebranded **Azure AI Foundry → "Microsoft Foundry."** **The SDKs, package names, ARNs, and most docs URLs still say the old names** (`google-cloud-aiplatform`, `azure-ai-foundry`, `aws ... bedrock`). Throughout this chapter we use the SDK/old name first and the new marketing name in parentheses, because the old name is what you'll type.

---

## Part A — The one mental model: CONSUME vs HOST

**1. Every cloud forces the same split: consume a provider model, or host your own.**
Strip the branding away and every cloud GenAI catalog is two piles. **(a) CONSUME** — call a model the provider already runs, over an API, with **zero infrastructure on your side**: no instances, no GPUs, no scaling to configure. You send tokens, you get tokens, you pay per token. This is **AWS Bedrock**, **Azure OpenAI / Microsoft Foundry (model inference)**, **GCP Vertex AI / Gemini API**. **(b) HOST** — take an open or fine-tuned model *you* chose and deploy it onto compute *you* size and pay for: you pick the GPU instance, the instance count, the autoscaling, and you own the idle time. This is **Amazon SageMaker**, **Azure ML**, **Vertex AI endpoints**. The same cloud sells you both, often one click apart, which is exactly why the catalog feels confusing.
- *Build consequence:* The first question for any service on the menu is "**is this consume or host?**" — because it decides everything downstream: who owns scaling, what the bill looks like, whether you can run a model the provider doesn't offer, and how much ops you signed up for. Answering it correctly resolves half the catalog confusion before you read a single pricing page.

**2. Consume = you rent the model; host = you rent the machine the model runs on.**
The billing unit is the tell. **Consume** bills per **token** (and sometimes per request/image/second of audio) — you never see an instance, never see "idle," never pick a GPU. **Host** bills per **instance-hour** (or per-second of compute, or per-request *on infra you provisioned*) — you are paying for a machine whether or not a request is in flight (unless it's scaled to zero, [Part C](#part-c--serverless-inference-economics-the-headline)). Consume's ceiling is "whatever models the provider offers and whatever rate limits they give you." Host's ceiling is "whatever you can fit on the hardware you're willing to pay for."
- *Build consequence:* If your sizing question is "how many input/output tokens × price × requests" you're in the consume world ([Chapter 8](08-deployment.md)'s cost math applies directly). If your sizing question is "how many GPU-hours and how big a GPU" you're in the host world ([Part D](#part-d--self-hosting-sizing-when-you-host-what-hardware)). Knowing which arithmetic to even *do* comes straight from this distinction.

**3. You usually want CONSUME — host only when consume can't.**
For engineers building *on top of* LLMs, **consume is the default and the right answer most of the time**: no GPUs to size, no idle to pay for, no model to keep patched, instant access to frontier models, and someone else owns the 2am scaling page. You move to **host** only for concrete reasons: (1) you need a **specific open/fine-tuned model** the provider doesn't offer in its consume catalog; (2) **data residency / isolation** rules forbid the data leaving to a shared inference endpoint; (3) at very high, very steady volume a reserved GPU is genuinely cheaper than per-token; or (4) you need **latency or control** (custom inference code, no per-token rate limits) you can't get from the shared API.
- *Build consequence:* Don't reach for SageMaker/Azure ML/Vertex-endpoints because it sounds more "real." Hosting is a cost and an ops burden you take on *only* when one of those four reasons applies. If none does, the consume API is less code, less money, and less to operate — write that down as your default and make the host case argue against it.

---

## Part B — Reading the confusing catalogs (per cloud)

**4. AWS: Bedrock (consume) vs SageMaker (host).**
**Amazon Bedrock** is the consume service: a single API (`bedrock-runtime` `InvokeModel` / `Converse`) over a catalog of provider models (Anthropic Claude, Meta Llama, Amazon Nova, Mistral, etc.), zero infra, per-token billing. **Amazon SageMaker** is the host service: you deploy a model (your own, or one from **SageMaker JumpStart**'s pre-packaged catalog) onto endpoints you size. SageMaker's distinctive endpoint types matter for sizing: **Serverless Inference** (scales to zero, [Part C](#part-c--serverless-inference-economics-the-headline)), **Multi-Model Endpoints** (many models behind one endpoint to share GPU cost), and **Asynchronous Inference** (queue long requests, scale to zero between bursts — great for big-document/batch work). Bedrock *also* has managed-RAG (Knowledge Bases) and managed-agents (Agents / AgentCore) grafts — see [Part G](#part-g--the-decision-and-where-ragagents-live).
- *Build consequence:* "Use Bedrock" and "use SageMaker" are not interchangeable — Bedrock is "call Claude with no infra," SageMaker is "I'm running my own Llama-3 fine-tune on a `g5` I sized." If someone hands you a vague "deploy it on AWS," your first clarifying question is which of these two they mean, because the work and the bill are nothing alike.

**5. Azure: Azure OpenAI vs Microsoft Foundry vs AI Services vs Azure ML.**
Azure is the most name-confusing cloud right now. **Azure OpenAI** (now "Azure OpenAI in Microsoft Foundry Models") is the consume service for OpenAI models — you create a **deployment** of a model (e.g. `gpt-4o`) in your tenant and call it; per-token, zero infra. **Microsoft Foundry** (formerly **Azure AI Foundry**) is the umbrella platform — model catalog (OpenAI *and* others), the **Foundry Agent Service** (the Assistants-API replacement, see the callout), evals, and tooling. **Azure AI Services** are the task-specific cognitive APIs (Vision/OCR, Speech, Document Intelligence, **Azure AI Search** for managed RAG) — also consume. **Azure ML** is the host service: deploy your own/open model to managed online or batch endpoints you size. So: OpenAI models → Azure OpenAI; broader catalog + agents + evals → Foundry; ready-made vision/speech/search APIs → AI Services; your own model on your own compute → Azure ML.
- *Build consequence:* When a teammate says "we'll use Azure AI," pin down *which* — Azure OpenAI (consume OpenAI models), Foundry (the platform + agents, and where new agent work must go given the 2026-08-26 Assistants retirement), AI Services (managed vision/speech/search), or Azure ML (host your own). They bill and behave differently; the rename to "Microsoft Foundry" makes this worse, not better, so confirm against the SDK name you're actually importing.

**6. GCP: Vertex AI (consume *and* host) — "Vertex saves time, SageMaker gives control."**
**Vertex AI** (now "Gemini Enterprise Agent Platform") is both: **consume** Gemini and 200+ Model-Garden models via API, *and* **host** your own model on **Vertex endpoints** you size. The industry shorthand is *"Vertex saves time, SageMaker gives control"* — Vertex bundles more for you out of the box: **BigQuery / Vertex AI Search grounding** (managed RAG against your data with little wiring), **AutoML**, and an opinionated, more-managed path. **SageMaker** gives you finer-grained control of the serving layer (Multi-Model Endpoints, Async Inference, custom containers) at the cost of doing more yourself. Neither is "better" — they trade time-to-ship against control.
- *Build consequence:* On GCP your consume/host split is *within one product*, so name the **mode** not just the product: "Vertex (consume Gemini API)" vs "a Vertex endpoint (host my model)." And use the time-vs-control axis to choose across clouds — if you want grounding-against-your-data with minimal plumbing, Vertex's managed grounding is the time-saver; if you need a custom serving topology, SageMaker's endpoint types give you the knobs.

**7. The names are moving under you in 2026 — code to the SDK, not the marketing.**
Recap of the rebrands so they don't trip your reading: **Vertex AI → "Gemini Enterprise Agent Platform"** (2026-04-22), **Azure AI Foundry → "Microsoft Foundry,"** and Azure OpenAI is folded under Foundry's branding. But the **SDK names, package names, ARNs, resource types, and most docs URLs still use the old names** — `google-cloud-aiplatform`, `azureml`/`azure-ai-foundry`, `aws bedrock`. A blog from 2025 and a console screen from 2026 may describe the *same service* with different names.
- *Build consequence:* When you're searching docs or wiring SDKs, **trust the import/ARN over the marketing page** — that's the name that won't lie to you about which service you're actually calling. Note the rebrand once in your team's runbook so nobody re-litigates "is Foundry the same as Azure OpenAI?" every sprint.

---

## Part C — Serverless inference economics (the headline)

**8. The three billing shapes: scale-to-zero vs always-on vs provisioned-concurrency.**
This is the single most decision-relevant idea in the chapter. For a **hosted** endpoint there are three ways the meter can run:
- **Serverless / scale-to-zero:** the platform spins capacity up on a request and down to nothing when idle. **$0 while idle** — you pay only for the compute a request actually uses (often **per-millisecond**). The cost: a **cold start** — the first request after idle waits seconds (sometimes 10s+ for a large model) while capacity warms.
- **Always-on / provisioned (dedicated instances):** you keep N instances running 24/7. **Predictable, low latency** (no cold start, ever) but you **pay for every idle hour** — the meter runs at 3am with zero traffic.
- **Provisioned concurrency (the middle path):** you pay a **warm-pool fee** to keep a set number of instances pre-initialized *behind a serverless model*, so those buy away the cold start, while traffic above the warm pool can still scale. You're buying off the cold-start penalty for a predictable baseline fee.
- *Build consequence:* You choose the shape from your **traffic pattern**, and the choice *is* the cost/latency tradeoff: spiky/low/dev traffic → serverless (eat the cold start, pay nothing idle); steady high traffic or latency-SLA → always-on (pay for idle, never cold); bursty-but-latency-sensitive → serverless + provisioned concurrency (pay a baseline to kill cold starts, scale above it). Naming your traffic shape is how you pick.

**9. The precise answer to "does it still cost money when it's idle?"**
This question comes up in every architecture review, and the precise answer depends on the exact mode:
- **Consume API (Bedrock/Azure OpenAI/Vertex)** — **no.** Per-token; zero requests = zero cost. Nothing to idle.
- **Serverless inference, plain (e.g. SageMaker Serverless with no provisioned concurrency)** — **no.** Scales to zero; $0 idle, you accept cold starts.
- **Serverless + provisioned concurrency** — **yes**, partially. You pay the warm-pool fee for the provisioned instances *even at zero traffic* (that fee is literally what removes the cold start); usage above the pool is still pay-per-use.
- **Always-on / dedicated instances** — **yes.** You pay for every running instance-hour regardless of traffic.
- *Build consequence:* This is a precise, defensible answer to give in a review — and a precise trap to avoid: someone enables provisioned concurrency to "fix latency" and is then surprised by a 24/7 bill on an endpoint that serves ten requests a day. State the mode explicitly in your design and the idle-cost answer follows mechanically.

**10. Per-millisecond billing changes the math for bursty workloads.**
Serverless inference (and serverless compute generally) often bills **per millisecond of actual compute**, not per provisioned hour. For a workload that's busy 2% of the day, per-ms billing means you pay for ~2% of the time instead of 100% — a serverless endpoint can be an order of magnitude cheaper than an always-on one *for that shape*. The crossover flips as utilization rises: past some steady-traffic threshold, the always-on instance's flat hourly rate beats paying per-ms for near-constant compute, and you should switch to always-on (or reserved capacity).
- *Build consequence:* Don't pick the shape by vibe — estimate **utilization** (what fraction of the day is the endpoint actually computing?). Low utilization → serverless/per-ms wins big. High, steady utilization → always-on wins. The deliverable makes you compute this crossover for a stated traffic shape rather than guessing.

---

## Part D — Self-hosting sizing (when you host, what hardware?)

**11. GPU instance selection: the practical tiers.**
When you host, you pick a GPU instance, and there are really three tiers you'll touch as an app engineer:
- **Inference / mid-tier: NVIDIA A10G** — AWS `g5`, GCP `g2`, Azure `NVadsA10` — ~24 GB VRAM, the workhorse for serving small-to-mid open models (7B–13B, quantized larger). Cheap, plentiful, usually no special quota fight.
- **Large inference / light training: A100** — AWS `p4d`/`p4de`, GCP `a2`, Azure `ND A100` — 40 or 80 GB VRAM, for big models or high throughput; expensive and quota-gated.
- **Frontier training: H100/H200** — `p5`, `a3`, `ND H100` — you almost certainly **don't** need these (you consume, you don't train); listed so you recognize them on the menu and don't accidentally request one.
- *Build consequence:* For serving an open/fine-tuned model you build on, **start at A10G (`g5`/`g2`/NVadsA10)** and only move up if the model doesn't fit in VRAM or you can't hit your throughput target. Reaching for A100/H100 by default is how a hosting bill 5–10×'s for no benefit.

**12. The VRAM → throughput → cost heuristic.**
A back-of-envelope you can do in your head: **the model's weights must fit in GPU VRAM, plus headroom for the KV cache (context).** Rough rule: a model needs about **2 GB VRAM per billion parameters at fp16**, ~**1 GB/B at int8**, ~**0.5 GB/B at int4**. So a 7B model ≈ 14 GB fp16 (fits an A10G's 24 GB with room for context); a 70B model ≈ 140 GB fp16 (needs multiple A100-80GB or aggressive quantization). **Quantization** is the main lever to fit a bigger model on cheaper hardware (at some quality cost — eval it, [Chapter 7](07-evaluation.md)). Then **throughput** (tokens/sec, concurrent requests) sets how many instances you need; **cost = instances × instance-hour rate × hours**.
- *Build consequence:* Before you request any GPU, do the VRAM arithmetic: does the model fit on the cheapest instance that holds it (after quantization)? That one calculation tells you the instance tier, and the instance tier × count × hours is the whole hosting bill. Get the fit wrong and you either OOM at deploy or over-provision a `p4d` you didn't need.

**13. Quotas are a real, slow gatekeeper — request them first.**
Cloud accounts start with **zero or near-zero GPU quota**, especially for A100/H100. You must file a **service quota increase** (AWS Service Quotas, Azure quota request, GCP quota increase) and approval can take **hours to days**, sometimes with a justification. A `g5`/A10G request is usually fast; an A100/H100 request can be slow or denied for new accounts.
- *Build consequence:* If your plan involves hosting on anything bigger than an A10G, **file the quota request on day one**, before you write the deploy code — otherwise "deploy the model" silently becomes "wait 2 days for AWS to approve a `p4d`." This is the most common avoidable schedule slip in a self-hosting project.

**14. Sizing a 2–3 hour GPU job (the concrete estimate).**
Putting it together for a bounded job (e.g. batch-embedding a corpus, or a few hours of serving for a demo): pick the instance from the VRAM fit (Part 12), look up its **on-demand hourly rate**, multiply by hours, and decide the shape. Anchored example: a single **`g5.xlarge` (1× A10G)** is roughly **~$1/hr on-demand** (varies by region — check live pricing); a 3-hour job is therefore **~$3** of GPU. The *same* model served always-on for a month (730 hrs) at that rate is **~$730/mo idle-or-not** — which is exactly why a bounded/bursty job belongs on serverless or a job that tears down when done, not an always-on endpoint left running.
- *Build consequence:* For any GPU work, write the estimate as **instance × $/hr × hours**, and pick serverless/teardown for bounded or bursty work so you're not paying ~$730/mo for an endpoint used 3 hours a week. The teardown step (Part hands-on) is not optional hygiene — it's the difference between a $3 experiment and a $730 surprise.

---

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
