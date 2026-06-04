## Advanced Track (Day 17+) — Cloud Managed-GenAI Platforms: choosing, sizing & costing a service

**Goal:** Be able to walk into any of the three big clouds (AWS, Azure, GCP), read its confusing GenAI catalog, and answer the two questions that actually matter for a build: *which* service do I use, and *what will it cost at my traffic?* By the end you can tell a "consume a provider model via API, zero infra" service apart from a "host your own model on instances you size" service on sight, explain serverless-vs-always-on-vs-provisioned inference economics (including the precise answer to "does it still cost money when idle?"), size a GPU job, and write a defensible managed-API-vs-self-host decision with a real monthly number attached.

**Why this matters:** Ch08 taught you to *operate* a service that calls a model — but it assumed you'd already picked the provider API and were calling it directly. In a real org you rarely get that clean choice. You're told "we're an AWS shop" or "everything must run in our Azure tenant," and now you have to map your Day 7 RAG bot or Day 9 agent onto **Bedrock or SageMaker**, **Azure OpenAI or Microsoft Foundry**, **Vertex or a Vertex endpoint** — names that overlap, rebrand, and bill in completely different shapes. Pick wrong and you either pay for idle GPUs you didn't need or hit a wall self-hosting a model you should have just called. The skill here is not "train a model" (you don't) — it's **catalog literacy + cost arithmetic**: which box on the menu is which, and what the meter does when no one is using it.

> **Setup assumed:** ch08 (Deployment) done — you have a deployable build and you understand client → backend → provider, streaming, retries, per-request cost logging. You have an account in at least one cloud (AWS / Azure / GCP) with billing enabled, the relevant CLI/SDK installed (`boto3`, `azure-ai-*`/`openai`, `google-cloud-aiplatform`), and *model access requested* (Bedrock model access, Azure OpenAI deployment, or a Vertex/Gemini project) — access approval can take hours to a day, so request it first. The hands-on can run almost entirely **local-first in Docker** for the self-hosting half; the managed-API calls hit real endpoints (cheap, pennies). This is a cloud-economics chapter, not an MLOps course — depth is "enough to choose and size," not "enough to run a GPU fleet."

**Suggested split:** Day 17 = Parts A–D (the consume-vs-host mental model, reading the catalogs, serverless inference economics, self-hosting sizing). Day 18 = Parts E–G (model registry & versioning, multi-region HA, the decision) plus the managed-RAG/agent cross-reference and the deliverable.

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
The billing unit is the tell. **Consume** bills per **token** (and sometimes per request/image/second of audio) — you never see an instance, never see "idle," never pick a GPU. **Host** bills per **instance-hour** (or per-second of compute, or per-request *on infra you provisioned*) — you are paying for a machine whether or not a request is in flight (unless it's scaled to zero, Part C). Consume's ceiling is "whatever models the provider offers and whatever rate limits they give you." Host's ceiling is "whatever you can fit on the hardware you're willing to pay for."
- *Build consequence:* If your sizing question is "how many input/output tokens × price × requests" you're in the consume world (ch08's cost math applies directly). If your sizing question is "how many GPU-hours and how big a GPU" you're in the host world (Part D). Knowing which arithmetic to even *do* comes straight from this distinction.

**3. You usually want CONSUME — host only when consume can't.**
For engineers building *on top of* LLMs, **consume is the default and the right answer most of the time**: no GPUs to size, no idle to pay for, no model to keep patched, instant access to frontier models, and someone else owns the 2am scaling page. You move to **host** only for concrete reasons: (1) you need a **specific open/fine-tuned model** the provider doesn't offer in its consume catalog; (2) **data residency / isolation** rules forbid the data leaving to a shared inference endpoint; (3) at very high, very steady volume a reserved GPU is genuinely cheaper than per-token; or (4) you need **latency or control** (custom inference code, no per-token rate limits) you can't get from the shared API.
- *Build consequence:* Don't reach for SageMaker/Azure ML/Vertex-endpoints because it sounds more "real." Hosting is a cost and an ops burden you take on *only* when one of those four reasons applies. If none does, the consume API is less code, less money, and less to operate — write that down as your default and make the host case argue against it.

---

## Part B — Reading the confusing catalogs (per cloud)

**4. AWS: Bedrock (consume) vs SageMaker (host).**
**Amazon Bedrock** is the consume service: a single API (`bedrock-runtime` `InvokeModel` / `Converse`) over a catalog of provider models (Anthropic Claude, Meta Llama, Amazon Nova, Mistral, etc.), zero infra, per-token billing. **Amazon SageMaker** is the host service: you deploy a model (your own, or one from **SageMaker JumpStart**'s pre-packaged catalog) onto endpoints you size. SageMaker's distinctive endpoint types matter for sizing: **Serverless Inference** (scales to zero, Part C), **Multi-Model Endpoints** (many models behind one endpoint to share GPU cost), and **Asynchronous Inference** (queue long requests, scale to zero between bursts — great for big-document/batch work). Bedrock *also* has managed-RAG (Knowledge Bases) and managed-agents (Agents / AgentCore) grafts — see Part G.
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
A back-of-envelope you can do in your head: **the model's weights must fit in GPU VRAM, plus headroom for the KV cache (context).** Rough rule: a model needs about **2 GB VRAM per billion parameters at fp16**, ~**1 GB/B at int8**, ~**0.5 GB/B at int4**. So a 7B model ≈ 14 GB fp16 (fits an A10G's 24 GB with room for context); a 70B model ≈ 140 GB fp16 (needs multiple A100-80GB or aggressive quantization). **Quantization** is the main lever to fit a bigger model on cheaper hardware (at some quality cost — eval it, ch07/ch13). Then **throughput** (tokens/sec, concurrent requests) sets how many instances you need; **cost = instances × instance-hour rate × hours**.
- *Build consequence:* Before you request any GPU, do the VRAM arithmetic: does the model fit on the cheapest instance that holds it (after quantization)? That one calculation tells you the instance tier, and the instance tier × count × hours is the whole hosting bill. Get the fit wrong and you either OOM at deploy or over-provision a `p4d` you didn't need.

**13. Quotas are a real, slow gatekeeper — request them first.**
Cloud accounts start with **zero or near-zero GPU quota**, especially for A100/H100. You must file a **service quota increase** (AWS Service Quotas, Azure quota request, GCP quota increase) and approval can take **hours to days**, sometimes with a justification. A `g5`/A10G request is usually fast; an A100/H100 request can be slow or denied for new accounts.
- *Build consequence:* If your plan involves hosting on anything bigger than an A10G, **file the quota request on day one**, before you write the deploy code — otherwise "deploy the model" silently becomes "wait 2 days for AWS to approve a `p4d`." This is the most common avoidable schedule slip in a self-hosting project.

**14. Sizing a 2–3 hour GPU job (the concrete estimate).**
Putting it together for a bounded job (e.g. batch-embedding a corpus, or a few hours of serving for a demo): pick the instance from the VRAM fit (Part 12), look up its **on-demand hourly rate**, multiply by hours, and decide the shape. Anchored example: a single **`g5.xlarge` (1× A10G)** is roughly **~$1/hr on-demand** (varies by region — check live pricing); a 3-hour job is therefore **~$3** of GPU. The *same* model served always-on for a month (730 hrs) at that rate is **~$730/mo idle-or-not** — which is exactly why a bounded/bursty job belongs on serverless or a job that tears down when done, not an always-on endpoint left running.
- *Build consequence:* For any GPU work, write the estimate as **instance × $/hr × hours**, and pick serverless/teardown for bounded or bursty work so you're not paying ~$730/mo for an endpoint used 3 hours a week. The teardown step (Part hands-on) is not optional hygiene — it's the difference between a $3 experiment and a $730 surprise.

---

## Part E — Managed model registry & versioning

**15. Native registries: SageMaker / Vertex / Azure ML model registry.**
Once you host (or fine-tune, ch06/ch11–12), you have **model versions** to track, and each cloud ships a native registry: **SageMaker Model Registry**, **Vertex AI Model Registry**, **Azure ML Model Registry**. They store model artifacts + metadata, version them, and — the part that matters for production — gate **deployment approval** (a registered model version must be *approved* before it can deploy to an endpoint). This is the host-world analog of ch08's "a prompt is a versioned, eval-gated, rollback-able artifact": here the *model* is the versioned, approval-gated artifact.
- *Build consequence:* If you host or fine-tune, your "which version is live and can I roll back?" question (ch08) is answered by the registry, and your eval gate (ch07/ch13) becomes the **approval gate** — a fine-tuned version doesn't reach an endpoint until it passes eval and gets approved. Wire the registry in from the first deploy, not after the first bad-model incident.

**16. Vendor-neutral MLflow — now covering GenAI apps, prompts & agents.**
**MLflow** is the cloud-agnostic alternative (and it runs everywhere, including self-hosted in Docker). Its **Model Registry** does versioning + stage transitions like the native ones, but it's portable across clouds. Importantly in 2026, MLflow has expanded beyond classic ML into **GenAI**: it now tracks **prompts** (a prompt registry with versions), **LLM/agent traces**, and **GenAI app evaluations** — so the same tool can version your prompts *and* your fine-tuned models *and* your agent traces, without locking you to one cloud's registry.
- *Build consequence:* If you're multi-cloud, want to avoid lock-in, or want one registry spanning prompts + models + agent traces, MLflow is the neutral choice — and it's the bridge between ch08's "prompts are versioned artifacts" and this chapter's "models are versioned artifacts," holding both in one place. If you're all-in on one cloud, the native registry is less to run.

---

## Part F — Multi-region HA as configuration

**17. High availability across regions is now a config flag, not a re-architecture.**
For **consume** services, the clouds turned multi-region resilience into a setting:
- **AWS Bedrock — cross-region inference profiles:** call an inference *profile* and Bedrock automatically routes/spreads the request across multiple regions, smoothing capacity and absorbing a regional throttle/outage without you managing failover.
- **Azure OpenAI — Global Standard (and Data Zone Standard) deployments:** a **Global Standard** deployment routes traffic to capacity across Microsoft's global fleet for the best availability/throughput; **Data Zone** variants keep routing within a geography for residency. You pick the deployment *type* and Azure handles the routing.
- **Vertex AI — auto-routing / global endpoints:** Vertex can route requests across regions for capacity and availability under one endpoint.
- *Build consequence:* For a consume workload you usually get HA by **choosing the right deployment type / inference profile**, not by building your own multi-region failover (which ch08 framed as a fallback you'd hand-roll). Know that the managed option exists and what it trades — Global/cross-region buys availability and throughput but can cross geographies, which **residency rules** may forbid, so pick the *data-zone-scoped* variant when residency matters.

**18. Self-hosted HA is still on you (and it's the expensive kind).**
The flip side: if you **host**, multi-region HA means **running your endpoint in multiple regions yourself** — multiplying the always-on instance cost by the number of regions, plus load balancing and health checks. There's no free "Global Standard" flag for your own SageMaker/Vertex endpoint; resilience costs real provisioned instances.
- *Build consequence:* HA cost is another entry on the consume-vs-host ledger: consume gets you cross-region resilience for ~a config choice; host makes you pay for (and operate) every replica region. If 99.9%+ availability is a requirement, that requirement pushes you *toward* consume unless you have a hard reason to host.

---

## Part G — The decision (and where RAG/agents live)

**19. Managed API vs self-host: the org usually decides, not the model.**
After all the catalog reading, the decision is rarely about the model itself — it's about **org context**:
- **Cloud commitment:** an org with a big AWS/Azure/GCP spend commitment (EDP/committed-use) will steer you to *that* cloud's GenAI service for discount and procurement reasons, full stop.
- **Infra integration:** the deciding factor is usually how the service plugs into the org's **IAM, VPC/network isolation, audit logging, and existing data connectors** — staying inside the tenant's identity and network boundary is often worth more than a marginally better model.
- **Residency / isolation:** if data can't leave a region or a tenant, that may *force* host (or a data-zone-scoped consume deployment) regardless of cost.
- **Then, and only then,** the model and price.
- *Build consequence:* Write the decision as: "We're an *X* shop, the service must sit inside our IAM/VPC and emit to our audit pipeline, residency requires *Y* → therefore *consume via Bedrock/Azure OpenAI/Vertex* (or *host on SageMaker/Azure ML/Vertex-endpoint* if a Part-A-3 reason applies)." The model name is the last line, not the first — and that's the realistic way these calls actually get made.

**20. Managed RAG and managed Agent services exist — but they're covered elsewhere (cross-reference).**
The clouds also sell **managed RAG** and **managed agents** as grafts on top of the consume services. Orienting summary only:
- **Managed RAG:** **Bedrock Knowledge Bases**, **Azure AI Search** (as the retrieval layer), **Vertex AI Search / RAG Engine** — they handle ingestion, chunking, embedding, the vector store, and retrieval so you don't wire it yourself (the time-saver from Part 6).
- **Managed agents:** **Bedrock Agents / AgentCore** (GA 2025-10-13), **Vertex Agent Engine**, **Microsoft Copilot Studio / Foundry Agent Service** (the Assistants replacement, retiring 2026-08-26).
- These are **covered in depth as grafts in the RAG chapter (ch04) and the Agents chapter (ch05)** — this chapter only places them on the catalog so you recognize them as "managed versions of things you built by hand," and know the two dates.
- *Build consequence:* When you see "Knowledge Base" or "Agent Engine" on a cloud's menu, recognize it as *managed RAG / managed agents* (a buy-vs-build choice you'll make in ch04/ch05), not a new primitive — and route the deep decision to those chapters. Here, just don't mistake them for something other than your Day 7 / Day 9 builds, managed.

---

**Side-by-side: calling a CONSUME model (zero infra) across the three clouds**

The code is short on purpose — that brevity *is* the point of "consume": no instance, no scaling, no GPU. (Anthropic's own API from ch01–08 is a fourth "consume" path; here we show it *through* each cloud's managed front door.)

```python
# AWS Bedrock — consume Claude via InvokeModel/Converse (boto3). Zero infra provisioned.
import boto3, json
brt = boto3.client("bedrock-runtime", region_name="us-east-1")
resp = brt.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",   # or a cross-region inference profile id (Part 17)
    messages=[{"role": "user", "content": [{"text": "One sentence: what is a cold start?"}]}],
)
print(resp["output"]["message"]["content"][0]["text"])
print(resp["usage"])  # inputTokens / outputTokens — this is your cost basis (ch08), no instance-hours

# Azure OpenAI (in Microsoft Foundry) — consume an OpenAI model you DEPLOYED in your tenant. Zero infra.
from openai import AzureOpenAI
client = AzureOpenAI(azure_endpoint="https://<resource>.openai.azure.com",
                     api_version="2024-10-21", api_key="<key-from-env>")
r = client.chat.completions.create(
    model="<your-deployment-name>",   # the deployment, e.g. a Global Standard gpt-4o (Part 17)
    messages=[{"role": "user", "content": "One sentence: what is a cold start?"}],
)
print(r.choices[0].message.content, r.usage)

# GCP Vertex AI (Gemini Enterprise Agent Platform) — consume Gemini. Zero infra.
import vertexai
from vertexai.generative_models import GenerativeModel
vertexai.init(project="<project>", location="us-central1")
m = GenerativeModel("gemini-1.5-pro")
out = m.generate_content("One sentence: what is a cold start?")
print(out.text, out.usage_metadata)
```

```python
# CONTRAST — HOST your own model on infra you size (SageMaker), one endpoint two ways.
# Note the things that DON'T appear in the consume code above: instance type, instance count, idle.
import boto3
sm = boto3.client("sagemaker")

# (a) SERVERLESS endpoint — scales to zero, $0 idle, accepts cold starts (Part C / Part 9).
sm.create_endpoint_config(
    EndpointConfigName="myrag-serverless",
    ProductionVariants=[{
        "VariantName": "v1", "ModelName": "my-jumpstart-llama",
        "ServerlessConfig": {"MemorySizeInMB": 6144, "MaxConcurrency": 10},  # no instance type, no idle bill
    }],
)

# (b) ALWAYS-ON endpoint — dedicated g5 (A10G, Part 11), predictable latency, PAYS FOR IDLE 24/7.
sm.create_endpoint_config(
    EndpointConfigName="myrag-alwayson",
    ProductionVariants=[{
        "VariantName": "v1", "ModelName": "my-jumpstart-llama",
        "InstanceType": "ml.g5.xlarge", "InitialInstanceCount": 1,   # ~$/hr running whether or not traffic
        # add ServerlessConfig.ProvisionedConcurrency on a serverless variant instead to buy off cold starts
        #   (Part 8/9) — that warm pool bills even at zero traffic.
    }],
)
# Cost intuition: (a) bills per-ms of actual compute (≈$0 at night); (b) bills g5.xlarge-hours all month.
```

---

**Resources** *(verify region/price on the live pages — prices move; this chapter gives shape, not a table)*
- **AWS** — Bedrock: https://aws.amazon.com/bedrock/ · Bedrock pricing: https://aws.amazon.com/bedrock/pricing/ · Cross-region inference: https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html · AgentCore (GA 2025-10-13): https://aws.amazon.com/bedrock/agentcore/ and https://aws.amazon.com/about-aws/whats-new/2025/10/amazon-bedrock-agentcore-available/ · SageMaker Serverless Inference: https://docs.aws.amazon.com/sagemaker/latest/dg/serverless-endpoints.html · SageMaker provisioned concurrency / async / multi-model: https://docs.aws.amazon.com/sagemaker/latest/dg/realtime-endpoints.html · SageMaker pricing: https://aws.amazon.com/sagemaker/pricing/
- **Azure** — Microsoft Foundry (was Azure AI Foundry): https://learn.microsoft.com/azure/ai-foundry/ · Azure OpenAI deployment types incl. Global Standard: https://learn.microsoft.com/azure/ai-foundry/openai/how-to/deployment-types · Assistants API retirement (2026-08-26) → Foundry Agent Service: https://learn.microsoft.com/azure/ai-foundry/openai/concepts/assistants · Azure AI Search (managed RAG): https://learn.microsoft.com/azure/search/ · Azure ML managed endpoints: https://learn.microsoft.com/azure/machine-learning/concept-endpoints
- **GCP** — Gemini Enterprise Agent Platform (was Vertex AI): https://cloud.google.com/products/gemini-enterprise-agent-platform · Vertex AI docs (SDK name still `aiplatform`): https://cloud.google.com/vertex-ai/docs · Vertex pricing: https://cloud.google.com/vertex-ai/pricing · Vertex AI RAG Engine: https://cloud.google.com/vertex-ai/generative-ai/docs/rag-engine/rag-overview · Vertex endpoints: https://cloud.google.com/vertex-ai/docs/predictions/overview
- **Vendor-neutral** — MLflow Model Registry: https://mlflow.org/docs/latest/model-registry.html · MLflow for GenAI (prompts/traces/agent eval): https://mlflow.org/docs/latest/llms/
- **Your own earlier days:** ch08 (this chapter's prerequisite — client→backend→provider, streaming, per-request cost), ch04 (managed RAG decision lives there), ch05/ch09–10 (managed agents decision lives there), ch06/ch11–12 (fine-tuning → registry approval gate), ch07/ch13 (eval as the approval gate).

**Hands-on tasks** *(Docker-first where a local backend runs; teardown is mandatory)*
1. **Consume, zero infra:** from a tiny backend (FastAPI/Flask in Docker), call **one** provider model via a managed API — **Bedrock `Converse`** *or* **Azure OpenAI** *or* **Vertex/Gemini**. Confirm in the console that **no instance/endpoint was provisioned** and note what you'd be billed on (tokens). Log input/output tokens (ch08).
2. **Host it, serverless:** deploy **one open model** — **SageMaker JumpStart serverless endpoint** *or* a **Vertex endpoint with min-replicas/scale-to-zero**. Hit it twice: once **after idle** (record the **cold-start latency**) and once immediately after (record the **warm latency**).
3. **Host it, always-on / provisioned:** redeploy the *same* model as an **always-on `ml.g5.xlarge` / provisioned-concurrency** endpoint. Hit it cold and warm again — note there's **no cold start** now.
4. **Record the three numbers that decide it:** for each of #2 and #3, write down (a) cold-start latency, (b) warm latency, (c) the **idle-cost difference** — serverless ≈ $0/hr idle vs always-on ≈ instance-$/hr × 730 hrs/mo. Use the **live pricing page** for the rate.
5. **Estimate the bill three ways** for a **stated traffic shape** (e.g. *2,000 requests/day, bursty 9–6 weekdays, ~800 in + 400 out tokens each*): (a) consume API (token math from ch08), (b) self-host serverless (per-ms/utilization), (c) self-host always-on (instance-hours). Produce a **real monthly $ figure** for each.
6. **TEARDOWN (mandatory):** delete both endpoints and any provisioned concurrency / always-on instances, and confirm in the console they're gone. *(Per Part 14, an always-on `g5` left running is ~$730/mo. Do not skip this.)* Stop local Docker containers too.
7. *(Stretch)* **Registry + approval gate:** register the hosted model version in the cloud's **Model Registry** (or MLflow in Docker), and wire a trivial **approval gate** (only an "approved" version may deploy) — connecting ch07/ch13 eval to deployment.
8. *(Stretch)* **HA as config:** switch the consume call to a **Bedrock cross-region inference profile** *or* an **Azure Global Standard** deployment and note what changed (one identifier) vs hand-rolling failover (ch08).

**Questions**

*Check understanding*
1. Define the **consume vs host** split, and give the billing unit and the "does it cost when idle?" answer for each.
2. Which of these is consume and which is host: **Bedrock, SageMaker, Azure OpenAI, Azure ML, Vertex (Gemini API), a Vertex endpoint**?
3. Name the three inference billing shapes and the cost/latency tradeoff of each.
4. Give the **precise** answer to "does it still cost money when idle?" for: consume API, plain serverless, serverless + provisioned concurrency, always-on.
5. State the VRAM heuristic (GB per billion params at fp16/int8/int4) and roughly what instance a 7B model needs vs a 70B model.
6. What are the two hard 2026 dates in this chapter, and what does each one mean for a build?
7. What's the "Vertex saves time, SageMaker gives control" tradeoff in one line each?
8. Why should you "code to the SDK name, not the marketing name" in 2026?

*Apply it*
9. A dev/demo endpoint serves ~20 requests/day in unpredictable bursts. Which billing shape, and what's the idle cost?
10. A latency-SLA chat feature does ~30 steady req/sec all day. Which shape, and why is plain serverless wrong here?
11. Your org has a large Azure commitment, data must stay in-region, and you need OpenAI models. Walk the Part-19 decision to a service + deployment type.
12. You must serve a **Llama-3-70B fine-tune** the provider doesn't offer in its consume catalog. Consume or host? What instance, and what do you check first?
13. Someone "fixed latency" by enabling provisioned concurrency on an endpoint that serves 10 req/day and is shocked at the bill. Explain exactly what happened.

*Stretch*
14. For a stated traffic shape, lay out the consume-vs-self-host-serverless-vs-self-host-always-on cost comparison and identify the **utilization crossover** where always-on starts winning.
15. Design multi-region HA for (a) a consume workload and (b) a self-hosted workload — and explain why HA requirements tend to push you *toward* consume.
16. You fine-tune a model monthly. Describe the registry + eval-gated approval + rollback flow, and where MLflow vs a native registry fits.

**Answer key**
1. **Consume** = call a provider-run model over an API with zero infra; bills per **token**; **no** idle cost. **Host** = deploy your own/open model on compute you size; bills per **instance-hour** (or per-ms on serverless); idle cost **depends on the mode** (yes for always-on/provisioned, no for scale-to-zero).
2. Consume: Bedrock, Azure OpenAI, Vertex (Gemini API). Host: SageMaker, Azure ML, a Vertex endpoint. (Vertex the product does both — name the *mode*.)
3. **Serverless/scale-to-zero** — $0 idle, but cold-start latency. **Always-on** — predictable low latency, but pay for every idle hour. **Provisioned concurrency** — pay a warm-pool fee to remove cold starts on a baseline while still scaling above it.
4. Consume API → **no**. Plain serverless → **no** (scales to zero). Serverless + provisioned concurrency → **yes** (warm-pool fee bills at zero traffic). Always-on → **yes** (every instance-hour).
5. ~2 GB/B fp16, ~1 GB/B int8, ~0.5 GB/B int4 (+ KV-cache headroom). 7B ≈ 14 GB fp16 → fits a single A10G (`g5`/`g2`, 24 GB); 70B ≈ 140 GB fp16 → multiple A100-80GB or heavy quantization.
6. **Azure OpenAI Assistants API retires 2026-08-26** → build new agents on the Foundry Agent Service / Responses API, not Assistants. **Bedrock AgentCore GA 2025-10-13** → it's a real production option now, not a preview to hedge against.
7. **Vertex saves time** — more bundled/managed out of the box (BigQuery/Search grounding, AutoML), faster to ship. **SageMaker gives control** — finer-grained serving (Multi-Model, Async, custom containers) at the cost of doing more yourself.
8. Because the marketing names rebranded in 2026 (Vertex→Gemini Enterprise Agent Platform, Azure AI Foundry→Microsoft Foundry) but the SDKs/packages/ARNs/docs still use the old names — the import/ARN is the unambiguous identity of the service you're actually calling.
9. **Serverless / scale-to-zero** — eat the cold start on the rare burst, pay **~$0 when idle** (which is almost always). Provisioned concurrency or always-on would bill 24/7 for a near-empty endpoint.
10. **Always-on (dedicated instances)** — steady high traffic means near-100% utilization, so the flat instance-hour rate beats per-ms, and you get no cold starts to hurt the latency SLA. Plain serverless would impose cold-start latency on the first request after any idle gap and (at this utilization) likely cost more.
11. Large Azure commitment → stay on Azure; need OpenAI models → **Azure OpenAI (consume)**; data must stay in-region → choose a **Data Zone Standard** (not Global) deployment type for residency-scoped routing. Model name is the last line.
12. **Host** (the consume catalog doesn't offer it). 70B fp16 ≈ 140 GB → won't fit one A10G; quantize (int8≈70 GB / int4≈35 GB) to target fewer/cheaper GPUs, or use A100-80GB(s). **Check VRAM fit and file the GPU quota request first** (A100 quota can take days).
13. Provisioned concurrency keeps a warm pool of instances initialized 24/7 and **bills the warm-pool fee even at zero traffic** — so a 10-req/day endpoint now pays for an always-warm instance around the clock. The right fix for that traffic is plain serverless (accept the cold start, pay ~$0 idle).
14. Compute: consume = (in×price + out×price) × req/day × 30; self-host serverless ≈ per-ms × actual compute time (scales with utilization); self-host always-on = instances × $/hr × 730. Plot cost vs utilization: serverless wins at low utilization, always-on's flat rate wins past the **crossover** (the utilization where 730 instance-hours costs less than paying per-ms for that much compute). Consume wins when there's no host-only reason and volume isn't extreme/steady.
15. (a) Consume → use **cross-region inference profile (Bedrock) / Global or Data-Zone Standard (Azure) / Vertex auto-routing** — HA is ~a config choice. (b) Self-host → run the endpoint in **multiple regions yourself**, multiplying always-on instance cost + LB/health-checks. Because (b) is expensive and operationally heavy, a hard HA requirement pushes you toward consume unless a host-only reason forces it.
16. Fine-tune → register the new **model version** in the registry (SageMaker/Vertex/Azure ML, or MLflow for cross-cloud) → **eval gate** (ch07/ch13) must pass → mark version **approved** → deploy approved version to the endpoint → roll back by re-pointing the endpoint at the previous approved version. **MLflow** if multi-cloud / want prompts+models+traces in one neutral place; **native registry** if all-in on one cloud (less to operate).

**Deliverable:** a short **"cloud GenAI service decision"** package for your Day 7 RAG bot or Day 9 agent: (1) the build calling **one consume API** (Bedrock/Azure OpenAI/Vertex) from a tiny Docker backend, with a note proving **zero infra was provisioned**; (2) the same open model deployed **once serverless** and **once always-on/provisioned**, with a recorded table of **cold-start latency, warm latency, and idle-cost-per-month** for each; (3) a **one-paragraph decision** — *managed-API vs self-hosted, serverless vs provisioned* — for a **stated traffic shape**, with a **real monthly-cost estimate for each option** and the chosen one justified via the Part-19 framing (cloud commitment / IAM-VPC-audit integration / residency, then model); (4) proof of **teardown** (endpoints deleted, containers stopped).

**Daily update (DM to Ayush):** one line — which cloud/services you mapped your build onto and the headline number, e.g. *"RAG bot on Bedrock Converse (zero infra) vs SageMaker; serverless cold-start ~9s / warm ~600ms, always-on g5.xlarge ~$730/mo idle; at 2k req/day bursty the consume API wins (~$140/mo) — chose Bedrock for IAM/VPC fit; both endpoints torn down."*

**Time:** ~2 days. Day 17: Parts A–D (consume-vs-host, the three catalogs, serverless economics, GPU sizing) + tasks 1–4. Day 18: Parts E–G (registry/versioning, multi-region HA, the decision + RAG/agent cross-ref) + tasks 5–6 and the decision package. *(Budget time up front to request model access and any GPU quota — both can take a day to approve.)*
