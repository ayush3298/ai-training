## Advanced Track — Cloud Data Platforms: Azure / AWS / GCP

**Goal:** Map the three hyperscalers' data services onto the lifecycle you already know, so a service
name ("Glue," "Dataflow," "Synapse") immediately tells you *which stage it does*. Our default is
Azure; you should be able to translate a pipeline across all three.

**What we assume you know:** The whole lifecycle and its tools (Chapters 1–11). This chapter is a
translation layer, not new concepts — every cloud service is a managed version of something you've
already learned.

**Why this matters:** Cloud service names dominate the bank (~370 mentions across Azure/AWS/GCP) and
job postings list a specific cloud. The skill is *transfer*: recognize that Glue, Dataflow, and ADF
are "the same lifecycle stage, different logo," so you're not relearning from scratch per cloud.

> **Setup assumed:** conceptual; free tiers exist (Azure/AWS/GCP) if you want to click through one.
> The mental model matters more than any one console.

---

#### Core concepts

**1. Every cloud has the same lifecycle stages — learn the mapping, not 200 product names.**
The lifecycle (Chapter 1) is constant; each cloud has a service per stage. The translation table:

    Stage              Azure                    AWS                     GCP
    Object storage     ADLS / Blob              S3                      GCS
    Ingestion/ETL      Data Factory (ADF)       Glue                    Dataflow / Data Fusion
    Big-data compute   Databricks / Synapse     EMR / Glue (Spark)      Dataproc / Dataflow
    Warehouse          Synapse / Fabric         Redshift                BigQuery
    Streaming          Event Hubs               Kinesis / MSK           Pub/Sub
    Orchestration      ADF / Airflow (MWAA-like) MWAA (managed Airflow) Cloud Composer (Airflow)
    Lakehouse          Azure Databricks + Delta  Databricks/EMR + Delta  Databricks / BigLake

- *Build consequence:* When you hit an unfamiliar service, ask "which lifecycle stage?" and map it to
  the tool you know (Glue ≈ managed Spark ETL; Pub/Sub ≈ Kafka-ish stream; BigQuery ≈ serverless
  warehouse). That turns "I don't know AWS" into "I know the stages; here's the AWS name."

**2. Consume vs host: managed services vs running it yourself.**
Each stage offers a spectrum: fully **managed/serverless** (BigQuery, ADF, Glue — the cloud runs it,
you configure) vs **self-managed on their infra** (Spark on EMR/Dataproc you size and tune). Managed
trades flexibility and some cost for far less operational burden.
- *Build consequence:* Default to the managed/serverless option unless you need control the managed
  service won't give (specific Spark versions, custom tuning, cost at scale). "Managed vs self-host"
  is a real decision per stage — and usually managed wins for a small team.

**3. Serverless economics: does it cost when idle?**
The cost question that recurs (Chapter 10): **serverless** services (BigQuery on-demand, many Glue/
Dataflow modes, serverless SQL) **scale to zero** — you pay per use, nothing when idle — while
**provisioned** clusters (a running EMR/Dataproc/Redshift cluster) bill while up regardless of use.
- *Build consequence:* For spiky or low-frequency workloads, serverless (scale-to-zero) avoids paying
  for idle. For steady high-volume workloads, a right-sized provisioned cluster can be cheaper than
  per-use billing. Ask "does this cost money when nobody's using it?" to choose.

**4. Azure first (our default stack): ADF, Synapse/Fabric, ADLS, and Azure Databricks.**
Since the course defaults to Azure + Databricks:
  - **ADLS** — object storage (the lake).
  - **Azure Data Factory (ADF)** — managed orchestration + ingestion/ETL (connectors, copy
    activities, triggers on source updates).
  - **Azure Databricks** — managed Spark + Delta (our processing/lakehouse engine).
  - **Synapse / Microsoft Fabric** — the warehouse/analytics side; Purview for governance/lineage.
- *Build consequence:* On Azure, a common shape is ADF (ingest/orchestrate) → ADLS (land) → Databricks
  (transform to medallion Delta) → Synapse/Fabric or Databricks SQL (serve) → Power BI. Know this
  spine cold; it's the bank's "Azure + PySpark + ADF" pipeline answer.

**5. The same pipeline, three clouds — translate it.**
Take the capstone: ingest files → land → transform → warehouse → BI. Across clouds:
  - **Azure:** ADF → ADLS → Databricks/Delta → Synapse/Power BI.
  - **AWS:** Glue/Lambda → S3 → EMR or Glue Spark / Databricks → Redshift → QuickSight.
  - **GCP:** Dataflow/Data Fusion → GCS → Dataproc/Dataflow → BigQuery → Looker.
- *Build consequence:* You can move between clouds by swapping per-stage services, because the
  *architecture* (lifecycle + medallion) is portable. Interviewers love "now do it on AWS instead" —
  the right move is to re-map stages, not redesign.

**6. Lock-in, portability, and migrations.**
Managed services speed you up but create **lock-in** (BigQuery SQL ≠ Redshift ≠ Snowflake; proprietary
formats). Portability levers: open table formats (Delta/Iceberg) and open file formats (Parquet) so
your *data* isn't trapped; SQL/dbt that's mostly portable; Spark that runs anywhere. Migrations
(Teradata→Databricks, Oracle→Redshift, Hive→BigQuery — all in the bank) are mostly: move data to
object storage, re-point/translate transformations, validate parity.
- *Build consequence:* Keep data in open formats and transformations in portable SQL/Spark/dbt to cap
  lock-in. For a migration, the playbook is land-to-object-storage → translate transforms → reconcile
  outputs against the source — not a rewrite from scratch.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Describe an end-to-end pipeline using Azure services (ADF + Databricks/PySpark) and the
   optimizations you applied.
2. How did you use dbt, Data Factory, BigQuery, and Airflow together in one project?
3. Can we use PySpark for an Oracle → Redshift migration without Glue? What are the steps?
4. Convert a Hive query into BigQuery.
5. Describe a Teradata → Databricks migration: steps, stack, and considerations.
6. Explain BigQuery architecture and how it bills queries.
7. How can you trigger a pipeline in Data Factory when source data is updated?
8. Describe a streaming pipeline using Dataflow (or a real-time architecture on AWS).
9. Compare Microsoft Fabric and Databricks (or data fabric vs Databricks).
10. Describe implementing Azure Data Lake and integrating it with ADF — challenges faced.
11. What are the main GCP data services and their use cases for a data engineer?
12. Describe an AWS architecture using S3 and CloudWatch monitoring.
13. How do you analyze and optimize ADF ingestion pipelines for performance/reliability?
14. When would you choose serverless vs a provisioned cluster on your cloud?
15. How would you keep a multi-cloud-portable design (limit lock-in)?

_(cloud-service terms appear ~370 times in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** Take the capstone pipeline and write the per-stage service mapping for **all three
clouds** (a table like concept 5), then pick one cloud and write a paragraph on which stages you'd
make serverless vs provisioned and why (cost when idle). Note one lock-in risk and how open formats
mitigate it.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~3–4 hours.

---
