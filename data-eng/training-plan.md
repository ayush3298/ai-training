# Data Engineering Training Plan

A top-down program for the team to learn modern data engineering — start from the big picture
(the data lifecycle), then go deeper stage by stage. Each chapter ends with a short daily update
(a one-liner is fine).

## Who this is for

**The reader is a near-fresher who knows Python and not much else.** We assume you can write a
function, use lists and dicts, loop over things, and read a file. We do **not** assume you know
SQL, databases, the cloud, distributed systems, or what a "data warehouse" is. Every term is
defined the first time it appears, every new idea is anchored to Python you already know, and the
ramp is deliberately slow. Where something is taught later, we say so instead of assuming it.

We are training these engineers to **build and operate data platforms** — to design pipelines that
move, transform, store, and serve data reliably **on top of** managed engines and warehouses —
**not** to build a query optimizer, a distributed scheduler, or a storage engine from scratch.

That sets the depth: internals (how Spark plans a shuffle, how a columnar file is laid out, how a
warehouse prunes partitions) are covered only deep enough to make good engineering decisions and to
tune what we run. We never write our own execution engine. The real depth of this program is in
chapters 2–11 (SQL & modeling, storage, Spark, Databricks, ingestion, streaming, transformation,
orchestration, warehousing, quality). Every Foundations topic is taught through one lens:
**"how does this change a pipeline I'd build?"**

## Default platform

This course teaches on a **default stack** the way the AI plan defaults to Anthropic/OpenAI:
**Databricks + Apache Spark + Delta Lake**, with **Azure** as the default cloud. Alternatives
(Snowflake, BigQuery, Redshift; AWS Glue/EMR; GCP Dataflow) are shown **side-by-side** wherever the
concept is provider-shaped, so the team can transfer the skill — but the hands-on spine is one stack
so tasks accrete instead of scattering.

## Course spine

Chapters are numbered in reading order — work straight through 1 → 11, then the Capstone.
SQL & modeling come early (Chapter 2) so there's a concrete schema to load, transform, and serve in
later chapters.

| # | Chapter | Focus |
|---|---------|-------|
| 1 | [Foundations — The Data Engineering Lifecycle](chapters/01-foundations.md) | the mental model: generation → ingestion → transformation → storage → serving; batch vs streaming; ETL vs ELT |
| 2 | [SQL & Data Modeling](chapters/02-sql-and-modeling.md) | joins, window functions, query optimization; dimensional modeling (Kimball, star schema, SCD) |
| 3 | [Storage & File Formats](chapters/03-storage-and-formats.md) | row vs columnar, Parquet, the lakehouse; Delta vs Iceberg vs Hudi; partitioning |
| 4 | [Spark & Distributed Processing](chapters/04-spark.md) | jobs/stages/tasks, shuffle, skew, partition vs repartition, joins, caching, PySpark optimization |
| 5 | [Databricks & Delta Lake](chapters/05-databricks-delta.md) | ACID, time travel, schema evolution, medallion, Delta Live Tables, Unity Catalog |
| 6 | [Ingestion & Integration](chapters/06-ingestion.md) | batch vs streaming ingest, CDC, connectors, idempotency, schema drift |
| 7 | [Streaming & Kafka](chapters/07-streaming-kafka.md) | event-driven design, partitions/replication, consumer groups, Structured Streaming, exactly-once |
| 8 | [Transformation & dbt](chapters/08-transformation-dbt.md) | ELT in the warehouse, dbt models/tests/docs, Jinja, incremental models, lineage |
| 9 | [Orchestration with Airflow](chapters/09-orchestration-airflow.md) | DAGs, scheduling, dependencies, retries/backfills, idempotent tasks, sensors |
| 10 | [Warehousing & Serving](chapters/10-warehousing-serving.md) | Snowflake (tasks/streams, SCD2, COPY from cloud storage), BigQuery/Redshift, serving layers |
| 11 | [Data Quality & Reliability](chapters/11-data-quality.md) | tests, SLAs/SLOs, data contracts, observability, freshness, the DataOps loop |
| 12 | Capstone Project | _to be planned_ |

## Chapters

1. [Foundations: the data engineering lifecycle](chapters/01-foundations.md)
2. [SQL & data modeling: the language and the shapes](chapters/02-sql-and-modeling.md)
3. [Storage & file formats: from rows to the lakehouse](chapters/03-storage-and-formats.md)
4. [Spark & distributed processing: how the work actually runs](chapters/04-spark.md)
5. [Databricks & Delta Lake: the default platform up close](chapters/05-databricks-delta.md)
6. [Ingestion & integration: getting data in, reliably](chapters/06-ingestion.md)
7. [Streaming & Kafka: from batch to event-driven](chapters/07-streaming-kafka.md)
8. [Transformation & dbt: ELT you can test and trust](chapters/08-transformation-dbt.md)
9. [Orchestration with Airflow: scheduling the whole graph](chapters/09-orchestration-airflow.md)
10. [Warehousing & serving: where the data lands for use](chapters/10-warehousing-serving.md)
11. [Data quality & reliability: shipping pipelines with confidence](chapters/11-data-quality.md)
12. Capstone Project _(to be planned)_

### Advanced & Production Track

Extension chapters that close known gaps for engineers operating real data platforms,
drafted alongside the Capstone (the rest of the track is backlogged in _Gaps & roadmap_ below):

1. [Performance & Cost Engineering](chapters/adv-performance-cost.md)
2. [Architecture & System Design](chapters/adv-architecture-system-design.md)
3. [Security, Privacy & Governance](chapters/adv-security-privacy-governance.md)
4. [Cloud Data Platforms — Azure / AWS / GCP](chapters/adv-cloud-platforms.md)
5. [DataOps, CI/CD & Infrastructure-as-Code](chapters/adv-dataops-cicd.md)

## How each chapter is built

Every chapter follows the same shape so the team always knows where to look:

- **Goal** → **Why it matters** → a **setup-assumed** note
- **Suggested split** across two working sessions
- **Parts** (numbered concepts) — each concept ends with a *Build consequence:* line tying it
  back to a pipeline you'd actually build
- Side-by-side **default-stack (Databricks/Spark/Delta) vs alternative (Snowflake / BigQuery / dbt)**
  code where it applies
- **Resources** → **Hands-on tasks** → **Questions** (Check understanding / Apply it / Stretch)
  → **Answer key** → **Deliverable** → **Daily update** → **Time estimate**
- Core chapters (1–11) end with a **15-question interview drill** pulled from `data-eng-questions.md`

## How the depth was calibrated

The chapter weighting is not guessed — it tracks the **2,814 real data-engineering interview
questions** in `data-eng-questions.md` (extracted from the team question bank). Three areas dominate
the bank and get the deepest treatment (2–3 days each):

- **Spark internals & optimization** — jobs/stages/tasks, shuffle, skew, partitioning, join strategies
- **Databricks & Delta Lake** — ACID, time travel, medallion, DLT, Unity Catalog
- **SQL & dimensional modeling** — window functions, query tuning, star schema, SCD types

A solid second tier (≈1–1.5 days each): streaming/Kafka, dbt, Airflow, Snowflake/warehousing, storage
formats. Foundations, ingestion, and data quality round it out. Every day's drill draws from the
matching question clusters.

## Gaps & roadmap (planned additions)

The core spine (chapters 1–11) is the build target. Additions that close known gaps for engineers
**operating** real platforms land two ways, mirroring the AI plan: **grafts** (new Parts inside
existing chapters — purely additive) and **extension chapters** (the "Advanced & Production Track,"
alongside the Capstone).

> **v1 (current cohort):** append as below. **v2 (next cohort):** interleave at the *ideal slot* so
> performance tuning follows Spark and governance precedes serving.

> **Status:** the spine and question bank are the first build wave (marked 🔨 = next to author,
> ⬜ = planned). Nothing is drafted yet — this outline defines the target.

### New chapters — Advanced & Production Track

| Topic | Scope | v2 ideal slot |
|-------|-------|---------------|
| ⬜ Performance & Cost Engineering | Spark tuning at scale (AQE, broadcast thresholds, partition sizing, skew/spill diagnosis from the Spark UI); cluster sizing & autoscaling; photon/runtime choices; Delta layout (OPTIMIZE, Z-ORDER, liquid clustering, file compaction); cost attribution & FinOps for data platforms | after [Chapter 4](chapters/04-spark.md) |
| ⬜ Architecture & System Design | lakehouse vs warehouse vs lake; batch + streaming together (Lambda vs Kappa); the medallion as architecture; data mesh vs centralized platform; choosing storage/compute/serving for a workload; the compound-pipeline framing (sources + ingest + transform + serve + orchestration, not one big job) | between [Chapter 5](chapters/05-databricks-delta.md) and [Chapter 10](chapters/10-warehousing-serving.md) |
| ⬜ Security, Privacy & Governance | RBAC & data access control; PII handling, masking & row/column-level security; encryption at rest/in transit; the catalog as governance (Unity Catalog, lineage, audit); GDPR/right-to-be-forgotten in a data lake; compliance (HIPAA/SOC2) for data platforms; a written data threat model | before [Chapter 10](chapters/10-warehousing-serving.md) |
| ⬜ Cloud Data Platforms — Azure / AWS / GCP | the hyperscaler data menu side-by-side: Azure (ADF, Synapse, Fabric, ADLS) — the default; AWS (Glue, EMR, Redshift, S3, Athena); GCP (BigQuery, Dataflow, Dataproc, GCS); managed-vs-self-host; serverless data economics (does it cost when idle?); reading the catalogs | after [Chapter 10](chapters/10-warehousing-serving.md) |
| ⬜ DataOps, CI/CD & Infrastructure-as-Code | testing pipelines (unit/integration/data tests); CI/CD for dbt & Spark jobs; environment promotion (dev→staging→prod); IaC (Terraform) for data infra; deployment, blue/green for data; monitoring, alerting & on-call for pipelines; the incident loop | between Architecture & Capstone |

### Grafts into existing chapters (additive Parts, no renumbering)

| Chapter | New Part | Hands-on |
|---------|----------|----------|
| ⬜ [02 — SQL & Modeling](chapters/02-sql-and-modeling.md) | Query optimization by reading the plan | EXPLAIN a slow join; fix it with the right index/partition; measure before/after |
| ⬜ [02 — SQL & Modeling](chapters/02-sql-and-modeling.md) | Slowly Changing Dimensions in practice | implement SCD Type 2 on a customer dimension; prove history is preserved on update |
| ⬜ [03 — Storage](chapters/03-storage-and-formats.md) | Partitioning & file-size pathologies | reproduce the small-files problem; fix with compaction; measure scan time |
| ⬜ [04 — Spark](chapters/04-spark.md) | Diagnosing skew & spill from the Spark UI | build a skewed join; read the UI to spot it; fix with salting/AQE; re-measure |
| ⬜ [05 — Databricks](chapters/05-databricks-delta.md) | Time travel & schema evolution | corrupt a table, restore via time travel; evolve schema and prove old readers still work |
| ⬜ [05 — Databricks](chapters/05-databricks-delta.md) | Medallion end-to-end | bronze→silver→gold on one dataset with quality gates between layers |
| ⬜ [06 — Ingestion](chapters/06-ingestion.md) | Idempotent & incremental ingestion | re-run an ingest twice; prove no duplicates (merge/upsert on a key) |
| ⬜ [06 — Ingestion](chapters/06-ingestion.md) | Change Data Capture | capture inserts/updates/deletes from a source; apply them downstream |
| ⬜ [07 — Streaming](chapters/07-streaming-kafka.md) | Exactly-once & checkpointing | kill a streaming job mid-batch; restart; prove no loss/no dup via checkpoints |
| ⬜ [08 — dbt](chapters/08-transformation-dbt.md) | Incremental models & tests | convert a full-refresh model to incremental; add schema + data tests that fail loudly |
| ⬜ [09 — Airflow](chapters/09-orchestration-airflow.md) | Backfills & idempotent tasks | backfill a date range; prove tasks are safe to re-run |
| ⬜ [11 — Data Quality](chapters/11-data-quality.md) | Data contracts & freshness SLOs | define a contract; break it upstream; catch it before it reaches serving |

### Rollout order

Legend: ✅ drafted · 🔨 next to author · ⬜ planned.

1. 🔨 **`data-eng-questions.md`** — the deduped, conversation-grouped question bank from the 2,814
   data-eng rows. Built first because every chapter's drill and depth calibration keys off it.
2. 🔨 **Core spine, chapters 1–11** — authored in reading order, each on the default stack
   (Databricks/Spark/Delta + Azure), with alternatives side-by-side.
3. ⬜ **Day reflow** — reflow the spine into day-sized files (`days/`) like the AI plan, ~30–35 days,
   each core day ending with a 15-question drill.
4. ⬜ **Capstone** — end-to-end pipeline: ingest → medallion transform in Spark/Delta → orchestrate
   with Airflow → serve in a warehouse, with quality gates throughout.
5. ⬜ **Advanced & Production Track** + grafts — performance/cost, architecture, security/governance,
   cloud platforms, DataOps/CI/CD.
