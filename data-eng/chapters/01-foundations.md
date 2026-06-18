## Chapter 1 — Foundations: The Data Engineering Lifecycle

**Goal:** Build a precise, working mental model of what data engineering *is* — the stages a
piece of data moves through, the concerns that ride alongside every stage, and the handful of
ideas that actually change how you design and debug a pipeline. By the end you should be able to
explain, to a teammate, *why* a pipeline is shaped the way it is and what that forces you to do
as a builder.

**Why this matters:** Most outages and bad design choices in data platforms come from a wrong
mental model — treating a pipeline like a one-shot script, ignoring that it will re-run, or
storing analytical data the way you'd store transactional data. Get these eight ideas right and
most of the later material (Spark, Delta, modeling, orchestration, quality) becomes obvious
instead of mysterious.

> **Setup assumed:** none yet. This chapter is concepts plus free hands-on (SQL in the browser,
> or a local DuckDB/Postgres). The default stack we build on for the rest of the course —
> **Databricks + Apache Spark + Delta Lake on Azure** — gets stood up in Chapter 4 onward.

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: what data engineering is, the lifecycle, the undercurrents, and
  the two reliability/latency ideas (idempotency, batch vs streaming).
- **Session 2** — concepts 5–8: the storage and transformation decisions (ETL/ELT, OLTP/OLAP,
  the lakehouse, the schema tax) + the hands-on and questions.

Everything you need is taught right here — there's no book or video to go watch first.

---

#### Core concepts

**1. The data engineering lifecycle is the spine: generation → ingestion → transformation → storage → serving.**
Every pipeline you ever build is some path through five stages. Data is *generated* by a source
(an app's OLTP database, an API, a sensor, a log). You *ingest* it (pull or receive it). You
*transform* it (clean, join, aggregate, model). It lives in *storage* (a lake, a warehouse). And
you *serve* it to a consumer (a dashboard, an ML model, a reverse-ETL sync). Storage isn't a
single stage so much as the substrate the other four sit on top of.
- *Build consequence:* When you're handed a vague request ("we need a sales dashboard"), your
  first move is to locate it on the lifecycle: what's the *source*, how does data get *in*, what
  *transformation* makes it usable, where does it *land*, who *serves* from it? Naming the five
  stages turns a fuzzy ask into a concrete pipeline design.

**2. The undercurrents ride alongside every stage — they're not a final step.**
Five concerns cut across the whole lifecycle: **security** (who can see what), **data
management** (governance, quality, lineage, metadata), **DataOps** (monitoring, automation,
incident response), **orchestration** (scheduling the graph of tasks), and **software
engineering** (version control, testing, IaC). The classic beginner mistake is treating these as
"things we'll add later."
- *Build consequence:* You design for these from the first commit, not after the first incident.
  "Where do credentials come from? How will I know when this breaks? Is this task safe to
  re-run?" are questions you answer while building, not during the postmortem.

**3. A pipeline re-runs — so it must be idempotent.** *(The most important reliability idea today.)*
A batch pipeline is not a one-shot script; it runs on a schedule, gets backfilled, retries after
failures, and gets re-triggered when someone fixes a bug upstream. **Idempotent** means: running
the same task twice produces the same end state as running it once — no duplicate rows, no
double-counted revenue, no drift.

Concretely, the non-idempotent way and the idempotent way to load one day of data:

    -- NOT idempotent: re-running appends the same rows again → duplicates
    INSERT INTO sales SELECT * FROM staging WHERE day = '2026-06-18';

    -- Idempotent: re-running overwrites just that day → same end state every time
    DELETE FROM sales WHERE day = '2026-06-18';
    INSERT INTO sales SELECT * FROM staging WHERE day = '2026-06-18';
    -- (or a single MERGE on a key; or partition-overwrite)

- *Build consequence:* Design every task so re-running it is safe. The two workhorse patterns are
  **MERGE/upsert on a key** and **partition overwrite** (replace a whole day/partition rather
  than append into it). If you can't safely re-run a task, you don't have a pipeline — you have a
  time bomb that goes off on the first retry.

**4. Batch vs streaming is bounded vs unbounded data — and batch is the default.**
*Batch* processes a finite chunk of data on a schedule (every hour, every night): bounded input,
known size, you can see the whole thing. *Streaming* processes an unbounded, never-ending feed of
events as they arrive: you never see "all" the data, only a window of it. Streaming buys you
*latency* (seconds instead of hours) and costs you *complexity* (out-of-order events, late
arrivals, exactly-once semantics, always-on infrastructure).

A quick way to size the choice — what's the *cost of staleness*?

    If the consumer is fine with data that's an hour old → batch. (Almost everything.)
    If a minute of staleness costs money or safety       → consider micro-batch / streaming.
    If you "want it real-time" but can't name the cost   → you want batch. Build batch.

- *Build consequence:* Default to batch. Reach for streaming only when a *named* requirement
  forces it (fraud blocking, live ops, real-time personalization) — and budget for the operational
  weight that comes with it. "Real-time" is a requirement to justify, not a free upgrade.

**5. ETL vs ELT — where you transform, and why ELT won.**
Both move data from source to a destination; they differ on *when* transformation happens.
  - **ETL (Extract → Transform → Load):** transform *before* loading, on a separate compute box.
    The warehouse only ever sees clean, modeled data. This was necessary when storage and
    warehouse compute were expensive and coupled — you couldn't afford to land raw junk.
  - **ELT (Extract → Load → Transform):** load *raw* into cheap storage first, then transform
    *inside* the warehouse/lakehouse using its own compute. You keep the raw data, and
    transformation is just SQL/Spark against what you already loaded.

ELT won for most modern stacks for two structural reasons: **(a) storage got cheap** (object
storage is ~pennies/GB/month, so keeping raw data is fine), and **(b) compute decoupled from
storage** (you can throw elastic warehouse/Spark compute at transformation without it being
"always on"). Keeping the raw layer means you can re-transform when requirements change — you
didn't throw the source away.
- *Build consequence:* On the default stack, you build **ELT**: land raw into the **bronze**
  layer of a lakehouse, then transform forward (bronze → silver → gold) with Spark/dbt. The
  medallion architecture you meet in Chapter 5 is just ELT with named layers.

**6. OLTP vs OLAP — two opposite workloads, which is *why* we move data at all.**
This is the single fact that explains why data engineering exists as a discipline.
  - **OLTP (transactional):** the app's database. Many tiny reads/writes, row-oriented, optimized
    for "fetch/insert *this one order*." Postgres, MySQL, the operational store.
  - **OLAP (analytical):** the warehouse/lakehouse. Few huge scans, column-oriented, optimized for
    "sum revenue across *50 million orders* by region." Snowflake, BigQuery, Databricks SQL.

Running heavy analytics directly on the OLTP database is the classic sin: a single `GROUP BY`
over the orders table locks rows, starves the app, and pages the on-call engineer. So we **copy**
data from OLTP into OLAP-shaped storage — and that copy *is* the ingestion half of the lifecycle.

Why columnar wins for OLAP, with hand-checkable numbers. Say a table has 50 columns, 100 GB
total, evenly sized, and your query touches just 2 of them (`region`, `revenue`):

    Row store:    reads all 50 columns to get to 2     → ~100 GB scanned
    Column store: reads only the 2 columns you asked   → ~4 GB scanned   (2/50 of the data)

Same query, ~25× less I/O — before compression, which columnar also does far better because each
column holds one data type. That's the whole reason analytical stores are columnar.
- *Build consequence:* Never point a BI dashboard at the production OLTP database. Choose
  row-oriented stores for transactional apps and column-oriented stores for analytics — and
  recognize that the gap between them is the job: moving and reshaping data from one world to the
  other.

**7. Storage/compute decoupling and the lakehouse — why the modern stack looks like it does.**
In the old warehouse, storage and compute were welded together: to store more you bought more
compute, and the cluster ran (and billed) whether or not anyone queried it. The modern pattern
**separates them**: data sits in cheap object storage (ADLS / S3 / GCS) as open file formats,
and *independent, elastic* compute (Spark, a serverless SQL warehouse) reads it on demand and
scales to zero when idle.
  - **Data lake:** cheap object storage holding raw files. Flexible, but no transactions, no
    schema enforcement — easy to turn into a "data swamp."
  - **Data warehouse:** managed, structured, transactional, fast — but historically closed and
    pricier, and awkward for unstructured data.
  - **Lakehouse:** the lake's cheap open storage **plus** a transactional table layer (Delta /
    Iceberg / Hudi) that adds ACID, schema enforcement, and time travel on top of the files. You
    get warehouse guarantees on lake economics. This is the default stack's home turf.
- *Build consequence:* "Where does the data live?" has three answers with different trade-offs.
  Decoupled storage/compute is why you can keep all your raw data cheaply *and* only pay for
  transformation compute while it runs — and it's why the lakehouse (Chapter 5) is the spine we
  build on rather than a classic warehouse.

**8. The schema tax is always paid — schema-on-write vs schema-on-read decides when.**
Structure has to be imposed somewhere. **Schema-on-write** validates and shapes data *as it's
loaded* (a warehouse rejects a bad row at load time) — strict, safe, but rigid. **Schema-on-read**
dumps raw data in and imposes structure *when you query it* (a lake reads JSON and parses fields
at query time) — flexible, but the bill comes due later when a field silently changes type and
every downstream query breaks. There is no "no schema" option; there's only *when* you pay.
- *Build consequence:* The medallion pattern is a deliberate answer to this: **bronze** is
  schema-on-read (land raw, stay flexible, never lose data), **silver/gold** are schema-on-write
  (enforce types and contracts before anyone builds on them). You pay the schema tax on purpose,
  at the boundary you choose — not by accident, at 2 a.m. This seeds data modeling (Chapter 2).

---

#### Resources (optional — the chapter is self-contained)
- **DuckDB** (free, runs in the browser at shell.duckdb.org or locally) — the SQL playground for
  the hands-on below; no install or account needed.
- The free docs pages for the modern stack's vocabulary, read for shape not product pitch:
  Databricks' "What is a lakehouse?" and dbt's "Analytics Engineering" overview.
- Skip if you're short on time — none of these are required to do the hands-on or the questions.

---

#### Hands-on (free tools, no cloud account needed)

**A. Place a real pipeline on the lifecycle.**
1. Pick any data product you've seen (a sales dashboard, a "users who bought X" feature, a daily
   report email). Write one line per lifecycle stage: source → ingestion → transformation →
   storage → serving.
2. For each of the five **undercurrents**, write one sentence: what would security / data
   management / DataOps / orchestration / software engineering look like for *this* pipeline?

**B. Feel why analytics doesn't belong on OLTP.**
1. In DuckDB (or Postgres), create a `sales` table and load ~1–5 million synthetic rows (a quick
   `range()` + random values generator is fine).
2. Run a heavy analytical query (`SELECT region, SUM(revenue) FROM sales GROUP BY region`) and a
   tiny transactional one (`SELECT * FROM sales WHERE id = 12345`). Time both.
3. Write two sentences: which store *shape* (row vs columnar) each query wants, and why running
   the first one against your app's production DB would be a bad idea.

**C. Make a non-idempotent task, then fix it.**
1. Write a load step that does `INSERT INTO target SELECT ... WHERE day = '<d>'`. Run it twice.
   Count the rows — observe the duplicates.
2. Rewrite it idempotently (delete-then-insert for that day, or a `MERGE` on a key). Run it twice.
   Confirm the row count is identical to running it once.
3. Write one sentence on why a scheduler retrying a failed run makes idempotency non-optional.

**D. Batch vs streaming — name the cost of staleness.**
1. For three scenarios — a nightly finance report, a fraud-blocking check at checkout, a "trending
   products" widget — decide batch vs streaming.
2. For each, write the *cost of one hour of staleness* in one sentence. Notice that this single
   question, not the word "real-time," drives the answer.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. Name the five stages of the data engineering lifecycle, in order.
2. What are the five undercurrents, and why are they not a "final step"?
3. In one line, what does it mean for a task to be *idempotent*, and why does a pipeline need it?
4. What is the core difference between batch and streaming data?
5. What's the difference between ETL and ELT, and what changed in the world to make ELT the
   default?
6. What is the difference between an OLTP and an OLAP workload?
7. Why are analytical stores column-oriented rather than row-oriented?
8. What does a lakehouse add on top of a plain data lake?
9. What does "schema-on-read vs schema-on-write" decide, and where does each live in the
   medallion pattern?
10. Why does decoupling storage from compute matter for cost?

**Apply it (builder scenarios — answer in 2–3 sentences):**
11. A teammate proposes pointing the new executive dashboard directly at the production Postgres
    that runs the app, "to keep it simple." What goes wrong, and what do you propose instead?
12. Your nightly load is scheduled with a retry-on-failure policy. Last night it failed halfway,
    retried, and now revenue is double-counted for that day. Diagnose the root cause and give the
    fix.
13. A stakeholder insists a report must be "real-time." How do you turn that into an engineering
    decision, and what's your default if they can't name a cost of staleness?
14. Requirements for a transformation changed and you now need a field you dropped during loading
    six months ago. Why does an ELT/lakehouse design save you here, and what would a strict ETL
    design have cost you?
15. You're loading semi-structured JSON from an API whose schema occasionally changes silently.
    Where in the medallion layers do you stay schema-on-read, where do you enforce schema, and
    why?

**Stretch / discussion (optional):**
16. The lifecycle calls storage the substrate rather than a single stage. What breaks in your
    mental model if you treat "storage" as just one step between transform and serve?
17. Give one realistic case where the *right* answer is genuinely streaming, and articulate
    exactly which "cost of staleness" justifies the extra operational weight.

**Answer key (peek only after attempting):**
1. Generation → ingestion → transformation → storage → serving. · 2. Security, data management,
DataOps, orchestration, software engineering; they cut across *every* stage, so they're designed
in from the start, not bolted on. · 3. Running a task twice yields the same end state as running
it once; pipelines retry/backfill/re-trigger, so non-idempotent tasks corrupt data on re-run. ·
4. Batch processes a bounded, finite chunk on a schedule; streaming processes an unbounded,
never-ending event feed as it arrives. · 5. ETL transforms before loading (on separate compute);
ELT loads raw first then transforms in the warehouse — cheap storage + storage/compute decoupling
made ELT viable and dominant. · 6. OLTP = many tiny row-oriented reads/writes for an app; OLAP =
few huge column-oriented scans for analytics. · 7. Analytical queries touch a few columns over
many rows, so reading only those columns scans far less data and compresses better. · 8. A
transactional table layer (ACID, schema enforcement, time travel) over cheap open object storage
— warehouse guarantees on lake economics. · 9. *When* you impose structure: schema-on-read at
query time (flexible, bronze), schema-on-write at load time (strict, silver/gold). · 10. Cheap
storage holds everything always, while elastic compute scales to zero when idle — you stop paying
for a cluster that isn't querying. · 11. Heavy analytical scans lock rows and starve the app;
copy the data into an OLAP-shaped store (lakehouse/warehouse) and serve the dashboard from there. ·
12. The load appended rather than overwrote (non-idempotent); switch to MERGE-on-key or
partition-overwrite for that day so retries are safe. · 13. Ask for the cost of one hour (or one
minute) of staleness; if they can't name one, build batch — "real-time" without a named cost is a
batch requirement. · 14. ELT kept the raw bronze data, so you re-transform from source;
strict ETL discarded the un-modeled field at load, so it's gone — you'd need to re-ingest from the
source if it's even still available. · 15. Bronze stays schema-on-read (land raw JSON, never lose
data); enforce schema entering silver (typed, validated) so gold and all downstream consumers
build on a stable contract. · 16. Storage shapes cost, access patterns, and guarantees for every
other stage; treating it as one step hides choices like row-vs-columnar, lake-vs-warehouse, and
retention that actually determine whether the pipeline works. · 17. e.g. fraud blocking at
checkout — a stale decision lets a fraudulent transaction through, so seconds of latency have
direct monetary cost that justifies the always-on streaming infrastructure.

---

#### Interview drill (self-test)

Real questions pulled from the data-engineering question bank, scoped to this chapter's
foundations. Answer out loud or in a line each, then check yourself against the chapter.

1. What is a data warehouse, and how does it differ from a data lake?
2. Can we use both a data lake and a data warehouse in the same project? When would you?
3. Compare blob storage, a data lake, and Delta Lake.
4. Explain the bronze–silver–gold (medallion) layer architecture for a data lake.
5. Describe batch processing. When is it the right choice?
6. Explain the difference between streaming inserts and batch loads.
7. What is the difference between OLTP and OLAP systems?
8. Explain ETL vs ELT, and why ELT is common on modern cloud stacks.
9. Describe an end-to-end data pipeline you have built or would build, stage by stage.
10. What are the types of data sources, and how do structured, semi-structured, and unstructured
    data differ?
11. What does a data engineer do, and how is the role different from a data analyst or data
    scientist?
12. Design a pipeline to ingest data from OLTP systems into a data warehouse and serve BI
    dashboards.
13. How do you implement data quality checks in a pipeline, and what do you check?
14. A backend analytics table has 2 billion+ rows; the team wants it in a data lake and a 3-month
    slice in Postgres. How would you design that movement?
15. What SLAs/SLOs would you define for a data pipeline, and how would you measure and report
    reliability?

_(466 foundation-relevant questions matched in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** A short note — 6–8 bullets — titled *"What a data platform is, and what that
means for building one."* Each bullet = one concept + its practical implication. Attach your
answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~2–3 hours (reading + hands-on + questions).

---
