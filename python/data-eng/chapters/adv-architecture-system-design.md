## Advanced Track — Architecture & System Design

**Goal:** Learn to design a whole data platform, not just one pipeline — choose storage/compute/
serving for a workload, combine batch and streaming, pick lakehouse vs warehouse vs both, and answer
the open-ended "design a system for X" questions with a defensible structure.

**What we assume you know:** All of Chapters 1–11 — this chapter composes them. The lifecycle
(Chapter 1) is the skeleton every design hangs on.

**Why this matters:** "Design a data platform / pipeline for …" is the single most common senior
prompt in the bank (architecture/design terms appear ~200 times). Interviewers test whether you can
reason from requirements to components and defend trade-offs — not whether you memorized one
architecture.

> **Setup assumed:** conceptual. Bring the capstone (Chapter 12) as your worked example to extend.

---

#### Core concepts

**1. Design from requirements, not from tools — start with the lifecycle skeleton.**
The trap is naming tools ("I'll use Kafka and Spark") before understanding the need. Start every
design by extracting the requirements: **volume** (GB or TB/day?), **velocity** (batch cadence or
real-time?), **latency SLA** (fresh by 6 a.m., or sub-second?), **variety** (structured/semi/
unstructured?), **consumers** (dashboards, ML, APIs?), **retention** and **budget**. Then walk the
lifecycle — generation → ingestion → transformation → storage → serving — placing a component at each
stage that the *requirements* justify.
- *Build consequence:* Answer "design X" by stating the requirements first, then filling the
  lifecycle stages — so each component choice is traceable to a need. "Why Kafka?" should have an
  answer in the requirements (real-time, multi-consumer), not "it's what I know."

**2. Lakehouse vs warehouse vs lake — and "both" is a real answer.**
The core storage decision (Chapters 1, 3, 10):
  - **Lake** — cheap raw files; flexible, no guarantees. Rarely the whole answer alone.
  - **Warehouse** — managed, SQL-first, great for BI; pricier, structured.
  - **Lakehouse** — open files + transactions (Delta); engineering/ML + reliability at lake cost.
  - **Both** — lakehouse for engineering/ML/large processing, a warehouse for BI serving — extremely
    common.
- *Build consequence:* Choose by consumer and workload: SQL-first BI shop → warehouse-centric;
  heavy processing/ML + open data → lakehouse; mixed → both with gold copied/served to the warehouse.
  Stating *why* (consumers, governance, scale) is the answer, not the product name.

**3. Batch + streaming together: Lambda vs Kappa, and "mostly batch" reality.**
Most platforms have both a batch path and a streaming path. Two reference patterns:
  - **Lambda** — separate batch layer (accurate, slow) and speed/streaming layer (fast, approximate),
    merged at serving. Powerful but you maintain *two* codebases.
  - **Kappa** — one streaming path handles everything; reprocess by replaying the log. Simpler when
    streaming covers your needs.
  - **Reality** — default batch; add a streaming path only for the sources that need it (Chapter 1's
    cost-of-staleness), and converge both into one silver/gold model (Chapter 6).
- *Build consequence:* Don't reach for Lambda's dual pipelines unless you truly need both accurate-
  batch and fast-approximate views; prefer one path (usually batch, Kappa-style for streaming) and
  converge. Two codebases for the same data is a maintenance cost you justify, not a default.

**4. The medallion as architecture, and data mesh vs centralized.**
Within the platform, the **medallion** (bronze/silver/gold) is the organizing pattern (Chapter 5):
raw landing, conformed truth, business marts. At org scale, a second question appears:
  - **Centralized platform** — one team owns the pipelines; simple governance, can bottleneck.
  - **Data mesh** — domain teams own their data "as a product" with shared standards/platform;
    scales org-wide but needs strong governance and platform tooling.
- *Build consequence:* For one team/product, a centralized medallion platform is right. Consider data
  mesh only when many domains and teams make a central team the bottleneck — and only with the
  governance (contracts, catalog, standards) to keep "products" interoperable. Don't cargo-cult mesh
  onto a small org.

**5. Compound system thinking: the platform is sources + ingest + transform + serve + orchestration + observability.**
A robust design isn't one big job; it's components with clear seams: ingestion (with idempotency +
schema handling), storage layers, transformation (dbt/Spark), orchestration (Airflow), serving
(warehouse/BI), and the cross-cutting layers — quality, governance, monitoring (the undercurrents).
The bank's "fault-resilient system for 1000 events/min" wants exactly this enumeration: a buffer
(Kafka) for durability/backpressure, a processor with validation + dead-letter, idempotent sinks,
retries, and monitoring — each component justified by a failure it prevents.
- *Build consequence:* Present designs as labeled components with their responsibility and the
  failure each handles (buffer → durability/backpressure; dead-letter → poison records; idempotent
  sink → safe retries; monitoring → detection). That structure *is* the senior answer.

**6. Trade-offs are the answer — name them explicitly.**
Every design choice trades something: streaming buys latency at the cost of complexity; denormalizing
buys read speed at the cost of storage/consistency; managed services buy speed-to-build at the cost
of flexibility/lock-in; more partitions buy parallelism until small-files overhead. Senior answers
*state the trade-off and why this side*, and note what would change the decision (e.g. "batch now; if
the fraud team needs sub-minute, add a streaming path").
- *Build consequence:* For each major choice, say what you're trading and the condition that would
  flip it. A design that acknowledges its trade-offs and failure modes reads as senior; one presented
  as obviously-correct reads as junior.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Design an Azure platform ingesting 2 TB/day from 50 sources with a 30-minute SLA and 5-year
   retention — what's the architecture?
2. Design a pipeline to ingest from OLTP systems into a warehouse and serve BI dashboards.
3. Design a fault-resilient system for 1000 events/min with large JSON payloads into an ERP.
4. Describe an end-to-end data pipeline you designed and the challenges involved.
5. Can we use both a data lake and a data warehouse in a project? How would you split them?
6. Design a pipeline for multiple sources: batch, streaming, and real-time from Kafka.
7. Describe a real-time analytics architecture (e.g. on AWS, or with Flink).
8. Design a data lake holding both structured and semi-structured data.
9. Discuss architectural decisions you made designing scalable pipelines.
10. Design an ingestion pipeline that handles both batch and streaming data.
11. Design a system for high-frequency data (WebSockets, Kafka, Redis caching) with schema evolution.
12. Apart from medallion architecture, what other approaches are there?
13. How do you handle multiple fact and dimension tables with historical and live data needs?
14. What considerations matter when building a pipeline for many upstream master-data sources?
15. Compare a centralized data platform vs a data mesh — when does each fit?

_(architecture/system-design terms appear ~200 times in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** Pick one "design X" prompt above and produce a one-page design: the extracted
requirements, a lifecycle diagram with a justified component at each stage, the batch/stream and
storage choices with trade-offs, and the failure-handling components. Defend two of your choices and
name what would flip each.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~3–4 hours.

---
