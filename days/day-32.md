# Day 32 — Cloud Platforms, part 2 — registry, multi-region HA & the decision

> [← Day 31](day-31.md) · [All days](README.md) · [Day 33 →](day-33.md)

**Module:** Cloud Managed-GenAI Platforms  ·  **Time:** ~2.5 hrs

## Where we are

_Continues **Cloud Managed-GenAI Platforms**. Earlier days covered Parts A, B, C, D; today picks up where they left off._

---

## Part E — Managed model registry & versioning

**15. Native registries: SageMaker / Vertex / Azure ML model registry.**
Once you host (or fine-tune, [Chapter 6](06-customization.md)), you have **model versions** to track, and each cloud ships a native registry: **SageMaker Model Registry**, **Vertex AI Model Registry**, **Azure ML Model Registry**. They store model artifacts + metadata, version them, and — the part that matters for production — gate **deployment approval** (a registered model version must be *approved* before it can deploy to an endpoint). This is the host-world analog of [Chapter 8](08-deployment.md)'s "a prompt is a versioned, eval-gated, rollback-able artifact": here the *model* is the versioned, approval-gated artifact.
- *Build consequence:* If you host or fine-tune, your "which version is live and can I roll back?" question ([Chapter 8](08-deployment.md)) is answered by the registry, and your eval gate ([Chapter 7](07-evaluation.md)) becomes the **approval gate** — a fine-tuned version doesn't reach an endpoint until it passes eval and gets approved. Wire the registry in from the first deploy, not after the first bad-model incident.

**16. Vendor-neutral MLflow — now covering GenAI apps, prompts & agents.**
**MLflow** is the cloud-agnostic alternative (and it runs everywhere, including self-hosted in Docker). Its **Model Registry** does versioning + stage transitions like the native ones, but it's portable across clouds. Importantly in 2026, MLflow has expanded beyond classic ML into **GenAI**: it now tracks **prompts** (a prompt registry with versions), **LLM/agent traces**, and **GenAI app evaluations** — so the same tool can version your prompts *and* your fine-tuned models *and* your agent traces, without locking you to one cloud's registry.
- *Build consequence:* If you're multi-cloud, want to avoid lock-in, or want one registry spanning prompts + models + agent traces, MLflow is the neutral choice — and it's the bridge between [Chapter 8](08-deployment.md)'s "prompts are versioned artifacts" and this chapter's "models are versioned artifacts," holding both in one place. If you're all-in on one cloud, the native registry is less to run.

---

## Part F — Multi-region HA as configuration

**17. High availability across regions is now a config flag, not a re-architecture.**
For **consume** services, the clouds turned multi-region resilience into a setting:
- **AWS Bedrock — cross-region inference profiles:** call an inference *profile* and Bedrock automatically routes/spreads the request across multiple regions, smoothing capacity and absorbing a regional throttle/outage without you managing failover.
- **Azure OpenAI — Global Standard (and Data Zone Standard) deployments:** a **Global Standard** deployment routes traffic to capacity across Microsoft's global fleet for the best availability/throughput; **Data Zone** variants keep routing within a geography for residency. You pick the deployment *type* and Azure handles the routing.
- **Vertex AI — auto-routing / global endpoints:** Vertex can route requests across regions for capacity and availability under one endpoint.
- *Build consequence:* For a consume workload you usually get HA by **choosing the right deployment type / inference profile**, not by building your own multi-region failover (which [Chapter 8](08-deployment.md) framed as a fallback you'd hand-roll). Know that the managed option exists and what it trades — Global/cross-region buys availability and throughput but can cross geographies, which **residency rules** may forbid, so pick the *data-zone-scoped* variant when residency matters.

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
- These are **covered in depth as grafts in the RAG chapter ([Chapter 4](04-rag.md)) and the Agents chapter ([Chapter 5](05-agents.md))** — this chapter only places them on the catalog so you recognize them as "managed versions of things you built by hand," and know the two dates.
- *Build consequence:* When you see "Knowledge Base" or "Agent Engine" on a cloud's menu, recognize it as *managed RAG / managed agents* (a buy-vs-build choice you'll make in [Chapter 4](04-rag.md)/[Chapter 5](05-agents.md)), not a new primitive — and route the deep decision to those chapters. Here, just don't mistake them for something other than your [Chapter 4](04-rag.md) / [Chapter 5](05-agents.md) builds, managed.

---

**Side-by-side: calling a CONSUME model (zero infra) across the three clouds**

The code is short on purpose — that brevity *is* the point of "consume": no instance, no scaling, no GPU. (Anthropic's own API from Chapters 1–8 is a fourth "consume" path; here we show it *through* each cloud's managed front door.)

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

---

## Module wrap-up — hands-on, questions & deliverable

**Resources** *(verify region/price on the live pages — prices move; this chapter gives shape, not a table)*
- **AWS** — Bedrock: https://aws.amazon.com/bedrock/ · Bedrock pricing: https://aws.amazon.com/bedrock/pricing/ · Cross-region inference: https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html · AgentCore (GA 2025-10-13): https://aws.amazon.com/bedrock/agentcore/ and https://aws.amazon.com/about-aws/whats-new/2025/10/amazon-bedrock-agentcore-available/ · SageMaker Serverless Inference: https://docs.aws.amazon.com/sagemaker/latest/dg/serverless-endpoints.html · SageMaker provisioned concurrency / async / multi-model: https://docs.aws.amazon.com/sagemaker/latest/dg/realtime-endpoints.html · SageMaker pricing: https://aws.amazon.com/sagemaker/pricing/
- **Azure** — Microsoft Foundry (was Azure AI Foundry): https://learn.microsoft.com/azure/ai-foundry/ · Azure OpenAI deployment types incl. Global Standard: https://learn.microsoft.com/azure/ai-foundry/openai/how-to/deployment-types · Assistants API retirement (2026-08-26) → Foundry Agent Service: https://learn.microsoft.com/azure/ai-foundry/openai/concepts/assistants · Azure AI Search (managed RAG): https://learn.microsoft.com/azure/search/ · Azure ML managed endpoints: https://learn.microsoft.com/azure/machine-learning/concept-endpoints
- **GCP** — Gemini Enterprise Agent Platform (was Vertex AI): https://cloud.google.com/products/gemini-enterprise-agent-platform · Vertex AI docs (SDK name still `aiplatform`): https://cloud.google.com/vertex-ai/docs · Vertex pricing: https://cloud.google.com/vertex-ai/pricing · Vertex AI RAG Engine: https://cloud.google.com/vertex-ai/generative-ai/docs/rag-engine/rag-overview · Vertex endpoints: https://cloud.google.com/vertex-ai/docs/predictions/overview
- **Vendor-neutral** — MLflow Model Registry: https://mlflow.org/docs/latest/model-registry.html · MLflow for GenAI (prompts/traces/agent eval): https://mlflow.org/docs/latest/llms/
- **Your own earlier chapters:** [Chapter 8](08-deployment.md) (this chapter's prerequisite — client→backend→provider, streaming, per-request cost), [Chapter 4](04-rag.md) (managed RAG decision lives there), [Chapter 5](05-agents.md) (managed agents decision lives there), [Chapter 6](06-customization.md) (fine-tuning → registry approval gate), [Chapter 7](07-evaluation.md) (eval as the approval gate).

**Hands-on tasks** *(Docker-first where a local backend runs; teardown is mandatory)*
1. **Consume, zero infra:** from a tiny backend (FastAPI/Flask in Docker), call **one** provider model via a managed API — **Bedrock `Converse`** *or* **Azure OpenAI** *or* **Vertex/Gemini**. Confirm in the console that **no instance/endpoint was provisioned** and note what you'd be billed on (tokens). Log input/output tokens ([Chapter 8](08-deployment.md)).
2. **Host it, serverless:** deploy **one open model** — **SageMaker JumpStart serverless endpoint** *or* a **Vertex endpoint with min-replicas/scale-to-zero**. Hit it twice: once **after idle** (record the **cold-start latency**) and once immediately after (record the **warm latency**).
3. **Host it, always-on / provisioned:** redeploy the *same* model as an **always-on `ml.g5.xlarge` / provisioned-concurrency** endpoint. Hit it cold and warm again — note there's **no cold start** now.
4. **Record the three numbers that decide it:** for each of #2 and #3, write down (a) cold-start latency, (b) warm latency, (c) the **idle-cost difference** — serverless ≈ $0/hr idle vs always-on ≈ instance-$/hr × 730 hrs/mo. Use the **live pricing page** for the rate.
5. **Estimate the bill three ways** for a **stated traffic shape** (e.g. *2,000 requests/day, bursty 9–6 weekdays, ~800 in + 400 out tokens each*): (a) consume API (token math from [Chapter 8](08-deployment.md)), (b) self-host serverless (per-ms/utilization), (c) self-host always-on (instance-hours). Produce a **real monthly $ figure** for each.
6. **TEARDOWN (mandatory):** delete both endpoints and any provisioned concurrency / always-on instances, and confirm in the console they're gone. *(Per Part 14, an always-on `g5` left running is ~$730/mo. Do not skip this.)* Stop local Docker containers too.
7. *(Stretch)* **Registry + approval gate:** register the hosted model version in the cloud's **Model Registry** (or MLflow in Docker), and wire a trivial **approval gate** (only an "approved" version may deploy) — connecting [Chapter 7](07-evaluation.md) eval to deployment.
8. *(Stretch)* **HA as config:** switch the consume call to a **Bedrock cross-region inference profile** *or* an **Azure Global Standard** deployment and note what changed (one identifier) vs hand-rolling failover ([Chapter 8](08-deployment.md)).

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
16. Fine-tune → register the new **model version** in the registry (SageMaker/Vertex/Azure ML, or MLflow for cross-cloud) → **eval gate** ([Chapter 7](07-evaluation.md)) must pass → mark version **approved** → deploy approved version to the endpoint → roll back by re-pointing the endpoint at the previous approved version. **MLflow** if multi-cloud / want prompts+models+traces in one neutral place; **native registry** if all-in on one cloud (less to operate).

**Deliverable:** a short **"cloud GenAI service decision"** package for your [Chapter 4](04-rag.md) RAG bot or [Chapter 5](05-agents.md) agent: (1) the build calling **one consume API** (Bedrock/Azure OpenAI/Vertex) from a tiny Docker backend, with a note proving **zero infra was provisioned**; (2) the same open model deployed **once serverless** and **once always-on/provisioned**, with a recorded table of **cold-start latency, warm latency, and idle-cost-per-month** for each; (3) a **one-paragraph decision** — *managed-API vs self-hosted, serverless vs provisioned* — for a **stated traffic shape**, with a **real monthly-cost estimate for each option** and the chosen one justified via the Part-19 framing (cloud commitment / IAM-VPC-audit integration / residency, then model); (4) proof of **teardown** (endpoints deleted, containers stopped).

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
