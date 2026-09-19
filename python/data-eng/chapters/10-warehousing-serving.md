## Chapter 10 — Warehousing & Serving: Where the Data Lands for Use

**Goal:** Understand the cloud data warehouse — how Snowflake / BigQuery / Redshift store and serve
analytical data, their cost models, how you load and model data for serving, and how the final
"serving layer" (BI dashboards, reverse-ETL) consumes it. This is where the pipeline pays off:
someone actually reads the data.

**What we assume you know:** SQL & dimensional modeling (Chapter 2), storage/columnar/partitioning
(Chapter 3), the medallion gold layer (Chapter 5), and OLAP vs OLTP (Chapter 1).

**Why this matters:** The warehouse is the most-asked product area in the bank (Snowflake alone
appears ~100 times). It's where cost blows up ("fast in dev, slow and expensive in prod"), where
modeling decisions meet real dashboards, and where the whole pipeline is judged by whether the
business gets fast, correct numbers.

> **Setup assumed:** DuckDB for the hands-on (its SQL — window functions, aggregates — is the same
> you'll write in Snowflake/BigQuery). Free trials of Snowflake/BigQuery exist if you want the real
> thing, but every concept here is learnable on DuckDB.

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: the cloud warehouse, storage/compute & cost, loading, Snowflake.
- **Session 2** — concepts 5–8: BigQuery, modeling for serving, the serving layer/BI, cost &
  performance + hands-on.

---

#### Core concepts

**1. A cloud data warehouse is a managed, SQL-first OLAP store with separated storage and compute.**
A **data warehouse** is the classic home for analytics: a managed system you load modeled data into
and query with SQL, optimized for big aggregations (OLAP, Chapter 1). The modern cloud warehouses —
**Snowflake, Google BigQuery, Amazon Redshift, Azure Synapse/Fabric** — all share the Chapter 3
trait: **storage and compute are separated**, so you store cheaply and spin compute up/down per
query.
  - **Warehouse vs lakehouse:** both serve OLAP. The lakehouse (Databricks/Delta, our default) keeps
    data as open files you own; a warehouse is more managed and SQL-centric. Many real stacks use
    *both* — lakehouse for engineering/ML, a warehouse for BI serving — and yes, you can use a lake
    and a warehouse in one project (a common bank question).
- *Build consequence:* The warehouse is usually your **serving** store — the gold layer that BI and
  analysts hit. Choose it for SQL-first analytics and broad tool support; choose/keep the lakehouse
  for large-scale processing and open storage. Knowing they coexist (not either/or) is the
  architecture answer.

**2. The cost model is the thing to understand — because separated compute is billed differently per warehouse.**
This is where the bank's "fast in dev, slow and costly in prod" lives. Each warehouse bills compute
differently:
  - **Snowflake** — you run **virtual warehouses** (named compute clusters you size XS→4XL and
    **auto-suspend** when idle). Cost = size × time running. Bigger size = faster but pricier.
  - **BigQuery** — **serverless**: no cluster to size; you're billed (on-demand) **per byte
    scanned**. `SELECT *` over a huge table is expensive; selecting 2 columns is cheap.
  - **Redshift/Synapse** — provisioned clusters (with newer serverless options); cost tracks cluster
    size/uptime.
- *Build consequence:* "Fast in dev, slow/costly in prod" is almost always *more data scanned* in
  prod + a too-small (Snowflake) or scan-heavy (BigQuery) query. The levers: scan less (partition/
  cluster, select fewer columns — Chapter 3), right-size the warehouse, and auto-suspend idle
  compute. You must know *which* cost model you're on to tune it.

**3. Loading: get data in with COPY/bulk loads from cloud storage, not row-by-row inserts.**
Warehouses load fastest in **bulk from cloud object storage**: stage files (CSV/Parquet/JSON) in
S3/GCS/Azure Blob, then `COPY INTO` (Snowflake) / `LOAD`/external tables (BigQuery, Redshift) to
ingest them in parallel. The bank's "daily sales CSV from S3 → transform → Snowflake" is exactly
this: land files, COPY into a staging table, transform to the model. Row-by-row `INSERT` is for tiny
data only — it's catastrophically slow at scale.
- *Build consequence:* Bulk-load from staged files in cloud storage; never loop single-row inserts to
  load a big dataset. Match the staged file format to the warehouse (Parquet is ideal — typed and
  columnar, Chapter 3), and use the warehouse's COPY options (skip headers, error handling) rather
  than pre-cleaning in slow code.

**4. Snowflake specifics: virtual warehouses, time travel & fail-safe, micro-partitions, streams & tasks.**
The most-asked warehouse, so know its vocabulary:
  - **Virtual warehouses** — independent compute clusters; you can give ETL and BI *separate* ones so
    a heavy load doesn't slow dashboards. Auto-suspend/auto-resume control cost.
  - **Micro-partitions & clustering** — Snowflake auto-stores data in small columnar
    micro-partitions with min/max metadata (automatic data-skipping, Chapter 3); a **clustering key**
    helps on very large tables.
  - **Time Travel & Fail-safe** — Time Travel queries/restores data up to N days back (like Delta's,
    Chapter 5); **Fail-safe** is a further 7-day Snowflake-managed recovery window after that. (Both
    cost storage.)
  - **Streams & Tasks** — a **stream** tracks changes (CDC) on a table; a **task** runs SQL on a
    schedule. Together they build incremental pipelines (stream → task MERGEs changes into the
    target) — Snowflake's native version of Chapter 6's incremental/CDC.
- *Build consequence:* Separate virtual warehouses to isolate ETL from BI cost/performance; use
  streams+tasks for in-warehouse incremental processing; rely on Time Travel for recovery. These are
  the daily Snowflake tools an interviewer expects you to name.

**5. BigQuery specifics: serverless, pay-per-scan, partitioning & clustering.**
BigQuery removes the cluster entirely — it's **serverless**, allocating **slots** (compute units)
automatically. Because on-demand billing is **per byte scanned**, the optimization game is *scan
less*: **partition** tables (usually by date) and **cluster** by common filter columns so queries
prune/skip data (Chapter 3), and never `SELECT *` when you need three columns. (Note: you partition
on date/integer/ingestion-time — *not* arbitrary VARCHARs, a bank gotcha.)
- *Build consequence:* On BigQuery, every unnecessary column and unpartitioned scan is direct money.
  Partition by date, cluster by frequent filters, select only needed columns, and prefer
  materialized/aggregated tables for repeated dashboard queries. Same SQL as Chapter 2; the cost
  discipline is what's new.

**6. Modeling for serving: denormalized star schemas and aggregates, built for reading.**
The serving layer is the **gold** layer (Chapter 5), modeled as Chapter 2's **star schema** —
fact + dimensions — and often **further denormalized/aggregated** for BI ("denormalized OLAP datasets
for BI" is a bank phrase). Dashboards should read pre-shaped, pre-aggregated tables, not compute
giant joins live on every page load.
- *Build consequence:* Build gold tables/marts shaped like the questions the business asks (a star
  for ad-hoc slicing, pre-aggregated summary tables for fixed dashboards). Pushing heavy joins/
  aggregations into the dashboard tool instead of pre-computing them in the warehouse is the #1 cause
  of slow dashboards.

**7. The serving layer and BI: dashboards, tools, and reverse-ETL.**
Finally, consumption:
  - **BI tools** — Power BI, Tableau, Looker connect to the warehouse and visualize gold. Tool choice
    (Power BI vs Tableau vs Looker/QuickSight) is a real interview question; the honest answer is
    "fit to the org" — Power BI in Microsoft/Azure shops, Tableau for rich exploration, Looker for
    governed metrics-as-code.
  - **Performance for BI** — pre-aggregate, use the warehouse's result cache, and model star schemas
    the tool understands; an "optimize the dashboard" task is usually "pre-shape the data + scan
    less," not tweaking the chart.
  - **Reverse ETL** — pushing gold metrics *back* into operational tools (CRM, ads) so the business
    acts on them — the serving lifecycle stage closing the loop.
- *Build consequence:* Your pipeline's job is to hand the BI layer fast, correct, well-modeled gold
  tables; performance and clarity are won in the data model, not the dashboard. Match the BI tool to
  the org, and use reverse-ETL when operational systems (not just humans) need the results.

**8. Cost & performance: one optimization ladder across all warehouses.**
When a warehouse query/dashboard is slow or expensive, work the ladder:
  1. **Scan less** — select only needed columns; ensure partition/cluster keys match the filters
     (Chapter 3). Biggest win, every warehouse.
  2. **Pre-aggregate** — materialize the repeated heavy query into a gold/summary table or
     materialized view, so dashboards read a small result.
  3. **Right-size & isolate compute** — Snowflake: size the virtual warehouse appropriately,
     auto-suspend, separate ETL/BI warehouses. BigQuery: control scanned bytes / reservations.
  4. **Cache** — lean on result caching for repeated identical queries.
- *Build consequence:* "Optimize this slow/expensive warehouse workload" has a repeatable answer:
  reduce bytes scanned first, pre-aggregate repeated work, then tune compute — in that order. Guessing
  at warehouse size before reducing the scan is the common, expensive mistake.

---

#### Resources (optional — the chapter is self-contained)
- **DuckDB** for the hands-on (its window/aggregate SQL is Snowflake/BigQuery-compatible).
- Free trials: Snowflake (30-day) and BigQuery sandbox if you want the real cost dashboards.
- Free reads for shape: the Snowflake "key concepts & architecture" and BigQuery "how it works"
  overviews — skim for the cost models, not the API.

---

#### Hands-on (Python — DuckDB; SQL transfers to Snowflake/BigQuery)

**A. Top-N per group (a real bank question, Snowflake-compatible).** Find the top 2 customers by
total sales in each region:

    import duckdb
    con = duckdb.connect()
    con.execute("""CREATE TABLE sales AS SELECT * FROM (VALUES
      ('North','Mara',100),('North','Ito',250),('North','Lena',80),
      ('South','Ada',300),('South','Bo',150),('South','Cy',400)
    ) t(region, customer, amount)""")
    print(con.execute("""
      SELECT region, customer, total FROM (
        SELECT region, customer, SUM(amount) AS total,
               ROW_NUMBER() OVER (PARTITION BY region ORDER BY SUM(amount) DESC) AS rnk
        FROM sales GROUP BY region, customer
      ) WHERE rnk <= 2
      ORDER BY region, total DESC
    """).fetchall())

1. Run it; confirm you get the top 2 per region. Write one sentence on why this window-function
   pattern (Chapter 2) is the standard "top-N per group" answer.

**B. Pre-aggregate for serving.** Build a `gold_region_daily` summary table (region totals) from the
fact, then "serve" a dashboard query off the small summary instead of the raw fact. Write one
sentence on why a dashboard reading the pre-aggregated table is faster/cheaper than re-aggregating
the fact on every load.

**C. Reason about cost (no code).** (a) A Snowflake query is fast in dev (small data) but slow and
costly in prod — list three things you'd check. (b) On BigQuery, why does `SELECT *` cost more than
`SELECT region, amount`? (c) Why partition a big table by date rather than not at all?

**D. Reason about architecture (no code).** Sketch: lakehouse (Databricks/Delta) for engineering +
a warehouse for BI serving. Where does gold live, what does each BI tool connect to, and where would
reverse-ETL push results?

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. What is a cloud data warehouse, and how does it relate to the lakehouse?
2. Can a project use both a data lake and a warehouse? Give an example split.
3. How does Snowflake bill compute vs how BigQuery bills it?
4. What's the fast way to load a large dataset into a warehouse, and what's the slow anti-pattern?
5. What is a Snowflake virtual warehouse, and why separate ETL and BI ones?
6. What are Snowflake streams and tasks used for together?
7. Why is `SELECT *` a cost problem on BigQuery specifically?
8. How should gold tables be modeled for serving, and why pre-aggregate?
9. Name three BI tools and the rough "fit" for each.
10. List the warehouse cost/performance optimization ladder in order.

**Apply it (short scenarios — answer in 2–3 sentences):**
11. The same Snowflake query is fast in dev and slow/expensive in prod. What do you investigate, in
    order?
12. A Matillion load into Redshift takes ~4 hours. Name three levers you'd pull to speed it up.
13. A Power BI dashboard is slow because it joins and aggregates three raw tables on every page load.
    What's the data-engineering fix?
14. You must load daily sales CSVs from S3 into Snowflake and serve them. Outline the path from file
    to dashboard.
15. A team wants a date-filtered query on a 5 TB BigQuery table to be cheap. What two table-design
    choices make it so, and what query habit?

**Stretch / discussion (optional):**
16. How do Snowflake Time Travel and Fail-safe differ, and how do they compare to Delta time travel?
17. When would you serve BI from the lakehouse directly vs copying gold into a separate warehouse?

**Answer key (peek only after attempting):**
1. A managed SQL-first OLAP store with separated storage/compute; both warehouse and lakehouse serve
OLAP, the lakehouse keeping open files and the warehouse being more managed/SQL-centric. · 2. Yes —
e.g. lakehouse for engineering/ML, a warehouse for BI serving. · 3. Snowflake bills sized virtual
warehouses by uptime; BigQuery (on-demand) bills per byte scanned, serverless. · 4. Bulk COPY/load
from staged files in cloud storage (parallel); row-by-row INSERT is the slow anti-pattern. · 5. A
named compute cluster you size and auto-suspend; separate ones so a heavy ETL load doesn't slow BI
queries. · 6. A stream tracks table changes (CDC) and a task runs scheduled SQL — together they build
incremental in-warehouse pipelines. · 7. Billing is per byte scanned, so reading all columns scans
far more data than the few you need. · 8. As star schemas plus pre-aggregated summary tables so
dashboards read small, pre-shaped results instead of computing big joins live. · 9. Power BI (Microsoft/
Azure shops), Tableau (rich exploration), Looker (governed metrics-as-code). · 10. Scan less →
pre-aggregate → right-size/isolate compute → cache. · 11. More data scanned in prod and/or a too-small
warehouse or scan-heavy query; check data volume, partition/cluster usage and columns selected, and
warehouse size. · 12. Stage files and bulk COPY, load Parquet not CSV, increase/right-size compute, and
ensure sort/dist keys (Redshift) match the queries. · 13. Pre-aggregate into a gold/summary table (and
model a star) so the dashboard reads a small pre-computed result. · 14. Land CSVs in S3 → COPY INTO a
staging table → transform to a star/gold model → BI tool reads gold. · 15. Partition by date and
cluster by frequent filter columns, and select only needed columns (avoid SELECT *). · 16. Time Travel
restores data within a configurable window (like Delta's); Fail-safe is an extra ~7-day
Snowflake-managed recovery after Time Travel expires; Delta uses its transaction log + VACUUM
retention. · 17. Serve from the lakehouse when its SQL endpoint and tool support suffice and you want
one copy; copy to a warehouse when BI needs the warehouse's concurrency, tooling, or governance.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. A Snowflake query is fast in dev but slow and costly in prod — what do you investigate?
2. Describe how Time Travel and Fail-safe work in Snowflake.
3. Does Snowflake have a trigger-based mechanism (streams & tasks)?
4. Explain BigQuery architecture and how it bills queries.
5. Explain how clustering works in BigQuery and how it complements partitioning.
6. Can you partition a BigQuery table on a VARCHAR column — and should you?
7. Design an ETL pipeline to read daily sales CSVs from S3, transform, and load into Snowflake.
8. A Matillion pipeline into Redshift takes ~4 hours — how do you optimize it?
9. Find the top two customers by total sales in each region (Snowflake-compatible query).
10. Can we use both a data lake and a data warehouse in a project? How would you split them?
11. Compare Tableau and Power BI (and when you'd pick Looker/QuickSight).
12. Describe designing denormalized OLAP datasets / data models for BI (Power BI).
13. Can you use Spark to connect to and query warehouse tables?
14. Steps to migrate Oracle → Redshift using PySpark (without Glue).
15. Explain BigQuery MERGE and how you'd use it for upserts.

_(Snowflake ~103, BigQuery ~66, BI/dashboards ~56, Redshift/Synapse ~47, warehouse ~33,
materialized-view/stored-proc ~25 questions in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** A DuckDB script with (1) a top-N-per-region query using a window function and (2) a
pre-aggregated gold summary table that a "dashboard" query reads. Plus a one-paragraph answer to
"why is the same query fast in dev but slow/expensive in prod, and how would you fix it?" Attach your
answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~4–5 hours.

---
