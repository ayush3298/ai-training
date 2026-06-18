## Chapter 12 — Capstone Project: An End-to-End Pipeline

**Goal:** Build one complete data pipeline that exercises every chapter — ingest raw data, land it
in a medallion lakehouse, transform and model it, orchestrate it on a schedule, serve it to a
dashboard, and guard it with quality checks. This is where the course stops being chapters and
becomes a thing you built.

**What we assume you know:** Chapters 1–11. The capstone doesn't teach new concepts; it makes you
*assemble* them. If a step feels shaky, that's a pointer back to the chapter to re-read.

**Why this matters:** Interviews ask "describe an end-to-end pipeline you built" more than almost
anything else (it's all over the question bank). After this, you have a real answer — a project you
designed, with the trade-offs you chose and why.

> **Setup assumed:** the free/local tooling used across the course — Python, DuckDB, the `deltalake`
> package, dbt-duckdb — is enough to build a complete working version on your laptop. Stretch goal:
> rebuild it on **Databricks Community Edition** (Spark + Delta) with the default stack, and add
> **Airflow via Docker** for orchestration.

---

### The project

**Build a sales analytics pipeline for a fictional retailer**, end to end:

> Raw orders arrive two ways — a **daily CSV drop** of the day's orders (batch) and a **stream of
> order-status change events** (an order goes pending → shipped → delivered). Customers and products
> come from a slowly-changing reference source. The business wants a **dashboard**: daily revenue by
> region and product category, top customers, and current order status — fresh each morning, correct,
> and trustworthy.

You may swap the domain (rides, streaming media, IoT sensors) as long as it has: a batch source, a
change/event source, at least one dimension that changes over time, and an analytical question worth
serving.

---

### Requirements, mapped to the lifecycle (and the chapters)

**1. Ingestion (Chapters 6, 7)**
- Ingest the daily orders **incrementally** with a stored **watermark** (not a full reload each day).
- Handle the order-status **change events** as a stream (you may simulate the stream with the
  Chapter 7 partitioned-log model if you're not running real Kafka).
- Make ingestion **idempotent**: re-running a day must not duplicate rows.
- Land everything **raw in bronze** first (schema-on-read; never lose source data).

**2. Storage & modeling (Chapters 3, 5, 2)**
- Store all tables as **Delta** (Parquet + transaction log), organized **bronze → silver → gold**.
- In **silver**, clean/type/deduplicate; resolve order status to *current* state with the
  latest-per-key window pattern; enforce schema entering silver.
- In **gold**, build a **star schema**: a `fact_sales` plus `dim_customer`, `dim_product`,
  `dim_date`. Make `dim_customer` **SCD Type 2** (track region/city changes over time).

**3. Transformation (Chapter 8)**
- Implement the silver→gold transformations as **dbt models** connected by `ref()`, with at least
  the staging and marts layers.
- Use an **incremental** materialization for the fact table.

**4. Orchestration (Chapter 9)**
- Wire the steps into a **DAG**: ingest → build silver → build gold (dims can run in parallel, then
  fact) → refresh the serving table. Each task **idempotent** and parameterized by the run's date so a
  **backfill** is safe.

**5. Serving (Chapter 10)**
- Produce **pre-aggregated gold tables** for the dashboard's questions (daily revenue by
  region/category, top-N customers per region, current status counts).
- Build (or sketch, with the actual queries) the **dashboard** reading only gold — no heavy live
  joins on raw data.

**6. Quality & reliability (Chapter 11)**
- Add **quality checks** entering silver: not-null keys, unique business keys, accepted status values,
  referential integrity (every `fact.customer_id` in `dim_customer`), and a **volume** check.
- Decide per check whether failure **gates** the pipeline or **quarantines** bad rows.
- Define two **SLOs** (one freshness, one quality) and say how you'd measure them.

**7. Cross-cutting (Chapters 1, undercurrents)**
- Put it all in **git**. Write a short **README** with an architecture diagram (the lifecycle stages)
  and your key decisions.
- Note where **security/governance** would go (access control, PII) even if you don't fully implement
  it.

---

### Suggested milestones

1. **Design** — one page: the architecture diagram (source → bronze → silver → gold → dashboard), the
   star-schema sketch, which source is batch vs stream, and which dimension is SCD2. *(Chapters 1–2)*
2. **Ingest → bronze** — incremental batch load with a watermark + simulated change stream, both
   idempotent, landing raw. *(Chapters 6–7)*
3. **Bronze → silver** — clean, dedup, current-status resolution, schema enforced, quality checks.
   *(Chapters 5, 11)*
4. **Silver → gold** — dbt star schema with SCD2 dimension and incremental fact. *(Chapters 2, 8)*
5. **Orchestrate** — the DAG with dependencies, retries, idempotent backfill. *(Chapter 9)*
6. **Serve** — pre-aggregated gold + dashboard/queries; prove a re-run and a backfill are safe.
   *(Chapter 10)*
7. **Harden** — quality gates, SLOs, README + diagram, and a written "what I'd do at 100× scale."

---

### Deliverables

1. **A working repo** (git) that runs end to end on the free/local stack: ingestion → medallion Delta
   → dbt gold → a serving query/dashboard, with quality checks.
2. **An architecture README** with a lifecycle diagram and your decisions (batch vs stream where and
   why; which dimension is SCD2 and why; where checks gate vs quarantine; your two SLOs).
3. **Proof of correctness**: a demonstration that (a) re-running a day adds no duplicates
   (idempotency), (b) a backfill over a date range is safe, (c) a deliberately bad batch is caught by
   a quality check, (d) time travel / an older table version is recoverable.
4. **A 1-page "scale-up" note**: what changes when this is 2 TB/day from 50 sources with a 30-minute
   SLA — the exact bank question. (Hint: real Kafka, Spark on Databricks, partitioning/Z-ORDER,
   separate compute, monitoring.)

---

### Evaluation rubric (how to know it's good)

| Dimension | What "good" looks like |
|-----------|------------------------|
| **Correctness** | Numbers are right; re-runs/backfills don't duplicate; deletes/updates resolve to current state |
| **Idempotency** | Every task is safe to re-run (MERGE/partition-overwrite, date-parameterized) |
| **Modeling** | Clean star schema; SCD2 done correctly; gold pre-aggregated for the dashboard |
| **Reliability** | Quality checks at silver; gate/quarantine decided; retries; SLOs defined |
| **Clarity** | Readable dbt models + DAG; README explains the *why*, not just the *what* |
| **Trade-offs** | You can defend each choice (batch vs stream, gate vs quarantine, SCD1 vs SCD2) |

---

### Stretch goals (optional)
- Rebuild on **Databricks Community Edition** with real Spark + Delta; read the Spark UI for one job.
- Run **real Kafka** (Docker) for the status-change stream and a Structured Streaming consumer.
- Orchestrate with **real Airflow** (Docker) and perform an actual backfill.
- Add **Great Expectations** validation docs alongside the dbt tests.
- Implement **OPTIMIZE/Z-ORDER** on the fact table and measure the query speedup.

---

**Deliverable:** the repo + README + proof-of-correctness + scale-up note above.

**Daily update:** which milestone you reached and any blockers.

**Time:** ~12–16 hours (spread across several sessions) — this is a real build.

---
