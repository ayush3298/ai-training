## Chapter 6 — Ingestion & Integration: Getting Data In, Reliably

**Goal:** Learn how data actually gets *into* your platform — full vs incremental loads, pulling
from databases / files / APIs, Change Data Capture (CDC), and how to make ingestion idempotent and
resilient so a retry or a schema change doesn't corrupt anything.

**What we assume you know:** Python (loops, `requests`-style API calls), and Chapters 1–5 (the
lifecycle, bronze layer, MERGE/idempotency, Delta, the "latest row per key" window function).

**Why this matters:** Ingestion is the first stage of every pipeline and the one most likely to
break — sources go down, APIs paginate weirdly, networks hiccup, and schemas drift without warning.
Reliable ingestion is what separates a toy script from a production pipeline.

> **Setup assumed:** Python with DuckDB for the hands-on. The patterns (watermarks, MERGE, dedup,
> retries) are engine-agnostic — the same logic runs in Spark/Databricks at scale.

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: full vs incremental, sources & connectors, APIs & pagination, CDC.
- **Session 2** — concepts 5–8: idempotent ingestion, schema drift, reliability, designing the
  pipeline + hands-on.

---

#### Core concepts

**1. Full load vs incremental load: re-pull everything, or only what changed.**
The first question for any source: each run, do you copy *all* of it or just the *new/changed* part?
  - **Full load** — re-read the entire source every time. Simple and self-correcting (you can't miss
    a change), but wasteful and slow — re-copying 2 billion rows nightly to get 1 million new ones is
    absurd.
  - **Incremental load** — read only rows newer than last time, tracked by a **watermark** (a
    "high-water mark": the max `updated_at` or id you've already loaded). Next run: `WHERE updated_at
    > :last_watermark`. Far cheaper, but you must track the watermark and handle late/updated rows.

- *Build consequence:* Default to incremental for anything large, keyed off a reliable
  `updated_at`/sequence column, and store the watermark durably (so a restart resumes correctly).
  Keep a periodic full reload as a safety net to catch anything the incremental logic missed. "Full
  load every night" on a big table is the most common avoidable cost in a pipeline.

**2. Sources and connectors: databases, files, and APIs — pull vs push.**
Data comes from three broad source shapes, and you either **pull** it (you go fetch on a schedule) or
have it **pushed** to you (it arrives as events):
  - **Databases** — pull rows over a connection (JDBC/ODBC). Big extracts should be chunked by key
    range or date so you don't lock the source or blow memory.
  - **Files** — land in blob storage / SFTP (CSV, JSON, Parquet). Ingest new files as they appear
    (Databricks **Auto Loader** does this idempotently — it tracks which files it has already seen).
  - **APIs** — pull JSON over HTTP (concept 3). Often the messiest.

Managed connector tools (Fivetran, Azure Data Factory, Kafka Connect, Airbyte) handle a lot of this
plumbing for common sources so you don't hand-write every extractor.
- *Build consequence:* Match the ingestion method to the source: chunked pulls for big DBs,
  file-arrival ingestion for blob/SFTP, paginated polling for APIs. For common sources, a managed
  connector is usually cheaper than maintaining bespoke extraction code — build custom only when the
  source is unusual.

**3. APIs and pagination: fetch JSON in pages, handle unknown counts, rate limits, and retries.**
A REST API rarely hands you everything at once — it **paginates** (returns 100 rows + a pointer to
the next page). You don't always know how many pages exist, so you loop until there are none left —
exactly a Python `while`:

    page, rows = 1, []
    while True:
        resp = get(f"/orders?page={page}")     # one page
        batch = resp["data"]
        if not batch:                           # empty page → we're done
            break
        rows.extend(batch)
        page += 1
    # cursor-style APIs: follow resp["next_cursor"] until it's null, instead of page numbers

Two realities: APIs **rate-limit** (HTTP 429 = "slow down") so you back off and retry; and responses
are nested JSON, which you flatten into rows for bronze.
- *Build consequence:* Never assume a fixed page count — loop on "is there a next page/cursor?" and
  handle 429s with backoff (concept 7). Store the raw JSON in bronze before flattening, so a parsing
  bug doesn't lose the original data (Chapter 1's "keep the raw" rule).

**4. Change Data Capture (CDC): capture inserts/updates/deletes from a source, instead of reloading.**
How do you keep a copy of a constantly-changing source table in sync *without* full reloads and
*without* missing updates and deletes? **CDC** captures the individual changes. Two flavors:
  - **Query-based CDC** — poll `WHERE updated_at > watermark` (concept 1). Simple, but it **misses
    hard deletes** (a deleted row has no `updated_at` to find) and needs a reliable timestamp.
  - **Log-based CDC** — read the source database's own **transaction log** (the redo/WAL the DB
    writes for every change). Tools like **Debezium** stream every insert/update/**delete** as an
    event, in order, with low load on the source. This is the robust option.

The output is a stream of change events (`op: insert/update/delete`, the row, a timestamp). You apply
them downstream with MERGE (Chapter 5) and reconstruct current state with "latest row per key"
(`ROW_NUMBER() OVER (PARTITION BY id ORDER BY event_time DESC) WHERE rn=1`, filtering out deletes) —
the exact pattern from Chapter 2.
- *Build consequence:* When you need deletes captured and low source load, use log-based CDC
  (Debezium) over query-based polling. The downstream pattern is always the same: land change events
  in bronze, then MERGE/dedup-to-latest into silver. (Bank trap: a CDC connector where "deletes
  aren't happening" usually means delete events aren't being captured or applied — check the source
  is emitting them and your MERGE has a `WHEN MATCHED ... DELETE` branch.)

**5. Idempotent ingestion: re-running a load must never duplicate data.**
Ingestion *will* be retried — a network blip, a scheduler restart, a manual re-run. Chapter 1's rule
applies hardest here. The patterns:
  - **MERGE on a business key** (Chapter 5) — upsert so a re-delivered row updates rather than
    duplicates.
  - **Partition overwrite** — replace the whole day's partition rather than appending to it.
  - **Dedup on arrival** — if a source/stream delivers duplicates ("at-least-once"), keep the latest
    per key with the ROW_NUMBER pattern before writing.
- *Build consequence:* Pick a stable business key per source and make the load upsert/overwrite, not
  append. Then a retry is safe by construction — you never have to reason about "did this half-run?"
  This is the difference between a pipeline you trust and one you babysit.

**6. Schema drift: the source changes shape without telling you — design for it.**
Sources add columns, rename fields, change a type from int to string. If your ingestion assumes a
fixed shape, it breaks (or worse, silently mis-reads). Defenses:
  - **Land raw in bronze schema-on-read** (Chapter 1) — accept whatever arrives so you never *lose*
    data to a shape change.
  - **Enforce schema entering silver** (Chapter 5) — so a drift fails loudly at a controlled boundary
    instead of corrupting gold.
  - **Detect and alert** — compare incoming schema to expected; new column → maybe auto-add, type
    change → alert a human.
- *Build consequence:* Don't let the source's shape changes silently flow to dashboards. Absorb
  drift in bronze, gate it at silver, and alert on unexpected changes — so drift becomes a
  notification, not a 2 a.m. outage. (Debezium emits schema-change events for exactly this reason.)

**7. Reliability: transient failures are normal, so retry with backoff and isolate poison records.**
Networks and APIs fail intermittently — that's not exceptional, it's expected. Build for it:
  - **Retry with exponential backoff** — on a transient error (timeout, 429, 503), wait 1s, 2s, 4s,
    … and retry a bounded number of times, rather than hammering or giving up. Combined with
    idempotency (concept 5), retries are safe.
  - **Dead-letter** the records you *can't* process (malformed JSON, failing validation) to a side
    location instead of crashing the whole batch — so one poison record doesn't stop the other
    999,999.
  - **Checkpointing** — record progress (the watermark, the last file/offset) durably so a restart
    resumes instead of redoing or skipping.
- *Build consequence:* Wrap external calls in bounded retry-with-backoff, make the operation
  idempotent so retries don't duplicate, and dead-letter the unprocessable — three habits that turn
  "the pipeline died because the API hiccuped" into "it retried and carried on."

**8. Designing ingestion: batch + streaming together, and choosing the intake.**
Real platforms ingest from many sources at once, some batch, some streaming. A common design: stream
events through **Kafka** (Chapter 7) into bronze for low-latency sources, batch-pull databases/files
into bronze on a schedule, then converge both into the same silver tables. Choosing the intake for a
new source (a real bank question):
  - **Read replica of the source DB** — good for periodic pulls without loading the primary; doesn't
    give you a real-time change stream.
  - **Kafka / Kafka Connect** — good when you need a durable, replayable, real-time event stream that
    many consumers share; more infrastructure.
  - **Log-based CDC (Debezium → Kafka)** — when you need every change including deletes, in near-real
    time, with minimal source load.
- *Build consequence:* Choose by the requirement: periodic snapshot → replica pull; real-time shared
  stream → Kafka; every change incl. deletes, low source impact → CDC. Then land everything in bronze
  and converge in silver, so batch and streaming sources end up in one consistent model.

---

#### Resources (optional — the chapter is self-contained)
- **DuckDB** for the hands-on (watermark, MERGE, dedup).
- Free reads for shape: the Debezium "what is CDC" intro and the Databricks Auto Loader overview.
- A public paginated REST API (e.g. a free JSON placeholder API) if you want to practice the
  pagination loop for real.

---

#### Hands-on (Python — DuckDB)

**A. Incremental load with a watermark.**

    import duckdb
    con = duckdb.connect()
    con.execute("CREATE TABLE source(id INT, updated_at INT, val TEXT)")
    con.execute("INSERT INTO source VALUES (1,100,'a'),(2,101,'b'),(3,102,'c')")
    con.execute("CREATE TABLE target(id INT, updated_at INT, val TEXT)")

    def incremental_load(watermark):
        rows = con.execute("SELECT * FROM source WHERE updated_at > ?", [watermark]).fetchall()
        con.executemany("INSERT INTO target VALUES (?,?,?)", rows)
        new_wm = con.execute("SELECT COALESCE(MAX(updated_at), ?) FROM target", [watermark]).fetchone()[0]
        return new_wm

    wm = incremental_load(0);  print("after first load, watermark:", wm)   # loads all 3
    con.execute("INSERT INTO source VALUES (4,103,'d')")                    # one new row arrives
    wm = incremental_load(wm); print("after second load, rows:",
          con.execute("SELECT COUNT(*) FROM target").fetchone()[0])        # 4, not 7

1. Run it; confirm the second load adds only the new row (4 total, not 7). Write one sentence on what
   the watermark is doing.

**B. Make ingestion idempotent (dedup to latest per key).** A source re-delivers `id=2` with a newer
`updated_at`. Append all raw arrivals to a `bronze` table, then build `silver` with the latest row
per id:

    con.execute("""CREATE TABLE silver AS
      SELECT id, updated_at, val FROM (
        SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) AS rn FROM bronze
      ) WHERE rn = 1""")

Insert a duplicate-but-newer `id=2` into bronze, rebuild silver, and confirm silver has one row per
id with the newest value. Write one sentence connecting this to "at-least-once delivery."

**C. Pagination loop (no real API needed).** Write a function `fetch_all()` that pulls from a fake
`get_page(n)` returning 100 rows for pages 1–3 and `[]` for page 4; loop until the empty page. Write
one sentence on why you loop on "empty page" rather than a fixed count.

**D. Reason about CDC (no code).** A source customer table gets inserts, updates, and *deletes*. (a)
Why does query-based CDC (`WHERE updated_at > wm`) miss the deletes? (b) How does log-based CDC
(Debezium) catch them? (c) Sketch how you'd apply the change stream to silver with MERGE.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. Full load vs incremental load — the trade-off, and what a watermark is.
2. Name the three broad source shapes and how you ingest each.
3. How do you page through an API when you don't know the number of pages?
4. What is CDC, and what's the difference between query-based and log-based?
5. Why does query-based CDC miss deletes, and what catches them?
6. Name two patterns that make ingestion idempotent.
7. What is schema drift, and where do you absorb it vs enforce against it?
8. What is retry-with-backoff, and why is it safe only with idempotency?
9. What does dead-lettering protect you from?
10. When would you choose Kafka over a read replica for a new source?

**Apply it (short scenarios — answer in 2–3 sentences):**
11. A nightly full reload of a 2-billion-row table takes hours to capture ~1M new rows. What do you
    change, and what must you store to make it work?
12. A Python ingestion job fails intermittently on network blips and sometimes leaves duplicates.
    Describe the two fixes you'd apply together and why both are needed.
13. A REST source returns nested JSON and an unknown number of pages, and 429s under load. Outline
    your ingestion loop.
14. A CDC connector is running but deletes aren't appearing downstream. What two things do you check?
15. The source team adds a new column without warning and your load breaks. How should the pipeline
    have been designed so this is a notification, not an outage?

**Stretch / discussion (optional):**
16. You need both a real-time event stream and periodic batch snapshots of the same source. Sketch an
    architecture that lands both into one silver model.
17. Why might you keep a periodic full reload even after building a correct incremental/CDC pipeline?

**Answer key (peek only after attempting):**
1. Full re-reads everything (simple, wasteful); incremental reads only rows past a watermark (the max
loaded `updated_at`/id) — cheap but you must track it. · 2. Databases (chunked pulls over JDBC),
files (ingest new files from blob/SFTP, e.g. Auto Loader), APIs (paginated HTTP polling). · 3. Loop
fetching pages until you get an empty page / null cursor. · 4. Capturing individual inserts/updates/
deletes; query-based polls a timestamp column, log-based reads the DB transaction log (Debezium). ·
5. A deleted row has no `updated_at` to find; log-based CDC reads the transaction log which records
the delete. · 6. MERGE/upsert on a business key, partition overwrite, or dedup-to-latest on arrival.
· 7. Source shape changing unexpectedly; absorb it raw in bronze (schema-on-read), enforce/validate
entering silver, and alert. · 8. Retrying transient failures with growing waits (1s,2s,4s…); safe
only if retries don't duplicate, i.e. the operation is idempotent. · 9. One unprocessable/poison
record crashing the whole batch — it's set aside instead. · 10. When you need a durable, replayable,
real-time stream shared by many consumers (a replica gives periodic pulls, not a change stream). ·
11. Switch to incremental on `updated_at`/id, storing the watermark durably; keep an occasional full
reload as a safety net. · 12. Retry-with-backoff (survive blips) + idempotent upsert/dedup (so
retries don't duplicate) — backoff handles the failure, idempotency handles the re-run. · 13. Loop
pages until empty/cursor-null, back off and retry on 429, store raw JSON to bronze, then flatten. ·
14. That the source emits delete events (log-based CDC enabled) and that your apply step has a
`WHEN MATCHED ... THEN DELETE` branch. · 15. Land raw in bronze (schema-on-read absorbs the new
column), enforce schema at silver (fails at a controlled boundary), and alert on schema changes. ·
16. Stream via Kafka into bronze + batch-pull snapshots into bronze, then MERGE/dedup both into the
same silver tables on a shared key. · 17. To self-heal any rows the incremental/CDC path missed
(missed events, watermark gaps, bugs) — a periodic full refresh reconciles the truth.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. After a full load completes, how do you enable CDC (e.g. in AWS DMS)?
2. Are there other ways to capture changes besides CDC?
3. Explain how Debezium handles schema drift.
4. A CDC source connector has performance issues and deletes aren't happening — what could be wrong?
5. For a new source, should you use Kafka or a read replica? Which and why?
6. Compare using Kafka vs a database table for data ingestion.
7. For a REST API returning JSON, how do you handle pagination when the page count is unknown?
8. Design a pipeline that fetches historical data plus daily incremental updates and exposes a query
   API.
9. How would you design robust retry, backoff, and idempotent mechanisms for a flaky Python pipeline?
10. Describe incremental sync from DB2/Cosmos change feed into a target using CDC or timestamps.
11. Explain your end-to-end architecture for ingesting files from SFTP.
12. Design an Azure platform ingesting 2 TB/day from 50 sources with a 30-minute SLA and 5-year
    retention.
13. Describe an ingestion pipeline that handles both batch and streaming data.
14. Why might a daily incremental load slow down for certain dates with no errors?
15. How do you use Kafka Connect to move a huge table into a lake (all of it) and Postgres (3 months)?

_(ingest ~58, incremental/watermark ~42, CDC/Debezium ~39, API/pagination ~24, connectors ~20
questions in the bank; 15 shown — more in [data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** A DuckDB script implementing an incremental load with a stored watermark and an
idempotent silver build (dedup-to-latest), proving a re-run adds no duplicates; plus a short note
choosing query-based vs log-based CDC for a source that has deletes, with the reason. Attach your
answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~4–5 hours.

---
