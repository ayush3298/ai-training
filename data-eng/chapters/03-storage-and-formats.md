## Chapter 3 — Storage & File Formats: From Rows to the Lakehouse

**Goal:** Understand *how* analytical data is physically stored — as files, in particular layouts —
and why those choices (columnar, compressed, partitioned, with a transaction log on top) are what
make a lakehouse fast and reliable. By the end you can pick the right file format for a job and
explain why a pile of CSVs is the wrong way to store a billion rows.

**What we assume you know:** Python (you've read/written CSV and JSON files), and Chapters 1–2
(OLTP vs OLAP, the columnar idea, the medallion layers).

**Why this matters:** In the cloud, "a table" is ultimately just *files in a folder*. The format of
those files decides your query speed, your storage bill, and whether two jobs writing at once
corrupt your data. Most "why is this so slow / so expensive" problems are storage-layout problems.

> **Setup assumed:** Python with DuckDB and pyarrow (`pip install duckdb pyarrow`). Everything runs
> on your laptop; the cloud storage we describe behaves the same, just bigger.

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: files as storage, row vs columnar, Parquet, compression.
- **Session 2** — concepts 5–8: partitioning, open table formats (Delta/Iceberg/Hudi), clustering,
  choosing a format + the hands-on.

---

#### Core concepts

**1. In the cloud, a "table" is just files in a folder — and the file format is a real decision.**
You already know how to save data to a file: `json.dump`, or write a CSV. A **data lake** is
literally that, at scale: files sitting in cloud **object storage** (Azure ADLS, AWS S3, Google
GCS) — think of it as an enormous shared folder you pay pennies per GB to keep. "Loading a table"
often means "writing some files"; "querying a table" means "an engine reads those files."
  - **CSV** — text, one row per line. Human-readable, universal — and terrible for analytics: no
    types (everything's a string), no compression, and to read one column you must read every byte
    of every row.
  - **JSON** — text, flexible/nested. Great for APIs, same problems at scale, plus it's bulky.

These are **row-oriented** (each record's fields stored together) and **untyped**. Fine for moving
data around; wrong for storing a lot of it to analyze.
- *Build consequence:* CSV/JSON are *interchange* formats — good for getting data in and out. The
  moment data lands to be queried repeatedly, you convert it to an analytical format (Parquet,
  below). Leaving a lake as raw CSV is a classic cost-and-speed mistake.

**2. Row storage vs columnar storage — the physical layout decides what you can skip.**
Chapter 1 said analytics stores are columnar; here's what that *means* on disk. Take three rows:

    id, country, revenue
    1,  US,      50
    2,  JP,      99
    3,  US,      20

  - **Row-oriented** writes them record by record: `1,US,50 | 2,JP,99 | 3,US,20`. To `SUM(revenue)`
    you must read all three fields of all three rows.
  - **Columnar** writes them column by column: `[1,2,3] [US,JP,US] [50,99,20]`. To `SUM(revenue)`
    you read *only* the revenue chunk and ignore the rest.

That's the 100 GB → 4 GB win from Chapter 1, now you can see why: the bytes you don't need aren't
even adjacent to the bytes you do.
- *Build consequence:* Columnar is faster for analytics (read few columns over many rows) and worse
  for "fetch one whole record" (you'd touch every column chunk) — which is exactly why apps stay
  row-oriented (OLTP) and analytics goes columnar (OLAP). The layout follows the workload.

**3. Parquet is the columnar workhorse: typed, compressed, and self-describing with stats.**
**Apache Parquet** is the default file format for analytical data. Over plain columnar it adds:
  - **Real types** — an `int` is stored as an int, not the text `"50"`. Smaller and no parsing.
  - **A schema baked in** — the file knows its own column names and types (self-describing), so you
    don't ship a separate schema.
  - **Column statistics per chunk** — each block records the min/max of each column, so an engine
    filtering `WHERE revenue > 1000` can *skip entire blocks* whose max is below 1000 without
    reading them. (This is called predicate pushdown / data skipping.)

Mentally: **CSV is a text dump; Parquet is a compressed, typed, indexed column store in a file.**
(ORC is a similar columnar format common in the Hadoop/Hive world; Avro is *row*-oriented and used
for streaming/messaging where you write whole records — know the names.)
- *Build consequence:* Store analytical data at rest as Parquet, not CSV — it's smaller (cheaper),
  faster to scan (column pruning + stats skipping), and carries its own types so downstream readers
  don't guess. You'll see your storage bill and query times drop by large factors from this one
  change.

**4. Compression is built in — and columnar compresses far better because like sits with like.**
Compression shrinks data by exploiting repetition. Columnar layout hands the compressor a gift:
within one column every value is the *same type and similar* — a `country` column is just
`US,US,JP,US,US…`, which crushes down to almost nothing (run-length/dictionary encoding). A row
layout interleaves an int, a string, a float, repeating — far less for the compressor to exploit.
Common codecs: **Snappy** (fast, the default), **Zstd**/**gzip** (smaller, slower).
- *Build consequence:* You'll routinely see Parquet land at a fraction of the equivalent CSV size,
  which directly cuts both storage cost and the bytes scanned per query (you pay for both in the
  cloud). Pick Snappy by default; reach for Zstd when storage cost matters more than write speed.

**5. Partitioning splits a table into folders by a column so queries skip most of the data.**
**Partitioning** physically divides a table's files into subfolders by the value of a column —
almost always a date:

    sales/
      day=2026-06-16/  part-0.parquet
      day=2026-06-17/  part-0.parquet
      day=2026-06-18/  part-0.parquet   ← a query for "today" reads ONLY this folder

A query with `WHERE day = '2026-06-18'` reads just that one folder and ignores the rest — called
**partition pruning**. This is the single biggest lever for big-table query speed, and it's why
incremental loads target one partition (and why partition-overwrite is the idempotent pattern from
Chapter 1).

The anti-pattern — **over-partitioning / small files**: partition by something high-cardinality
(like `user_id`) and you get millions of tiny folders each with a tiny file. Engines are slow at
opening huge numbers of small files (per-file overhead dominates), so the table gets *slower*, not
faster. Rule of thumb: partition by a low-cardinality column you actually filter on (date), and aim
for files in the hundreds-of-MB range, not KB.
- *Build consequence:* Choose the partition column by *how the table is queried* (almost always by
  date range), not arbitrarily. And watch file sizes: too-fine partitioning creates the small-files
  problem that compaction (Chapter 5) then has to fix.

**6. Open table formats (Delta / Iceberg / Hudi) add a transaction log on top of Parquet — turning a folder of files into a real table.**
A bare folder of Parquet files has a fatal flaw: **no transactions.** If two jobs write at once, or
one crashes mid-write, readers can see half-written, duplicated, or corrupt data — there's no
"commit." **Delta Lake, Apache Iceberg, and Apache Hudi** all solve this the same way: they keep the
data as Parquet files *plus a metadata log* that records, atomically, which files make up the table
*right now*. That log buys you:
  - **ACID transactions** — a write either fully commits or isn't visible at all; concurrent
    writes/reads stay consistent. (ACID = Atomicity, Consistency, Isolation, Durability — the
    "all-or-nothing, never half-done" guarantees.)
  - **Time travel** — because the log has the history of versions, you can query the table *as of*
    an earlier version ("yesterday's data") or roll back a bad write.
  - **Schema evolution** — add a column safely without rewriting everything or breaking old readers.

This is the difference between a *data lake* (just files) and a *lakehouse* (files + transactions +
schema). On our default stack the format is **Delta Lake**; Iceberg is the main open alternative
(strong on huge metadata and multi-engine use), Hudi is strong on streaming upserts. They're the
same idea with different trade-offs.
- *Build consequence:* Never build a production table as bare Parquet files you append to — one
  concurrent write or failed job corrupts it. Use a transactional table format so writes are
  all-or-nothing, you can audit/rollback via time travel, and schema changes don't break consumers.
  This is the foundation of Chapter 5.

**7. Clustering / sorting within partitions lets the engine skip even more — it complements partitioning.**
Partitioning skips whole *folders*; **clustering** (sorting data by a column within the files, e.g.
Delta's Z-ORDER or liquid clustering, BigQuery's clustering) makes the per-file min/max stats tight
so the engine skips *blocks within* a partition too. The bank's question — "partition by date,
cluster by `order_timestamp`, filter on `order_timestamp`" — answers: a filter on the *cluster* key
uses clustering (data-skipping within files), not partition pruning (which needs a filter on the
*partition* key). They stack: partition by the coarse thing you range-filter on (date), cluster by
the finer thing you also filter/join on.
- *Build consequence:* When a query filters on a column that *isn't* the partition key and is still
  slow, clustering/Z-ordering on that column is the fix — you reorganize the data so the engine
  reads fewer blocks, without changing the partition scheme.

**8. Choosing a format — a short decision ladder.**
  1. **Moving data in/out, or a human needs to read it?** → CSV/JSON (interchange).
  2. **Streaming/messaging, writing whole records?** → Avro (row-oriented, schema-friendly).
  3. **Analytical data at rest, read many times, not updated in place?** → Parquet (columnar).
  4. **A table you'll update, merge into, append to concurrently, or need history for?** → a
     transactional format (Delta on our stack; Iceberg/Hudi otherwise) — Parquet underneath, plus
     the log.
- *Build consequence:* Most pipelines use several at once: ingest CSV/JSON/Avro into **bronze**,
  store **silver/gold** as Delta tables. "What format?" is answered by "what happens to this data
  next?" — interchange, scan, or transactional table.

---

#### Resources (optional — the chapter is self-contained)
- **DuckDB + pyarrow** (`pip install duckdb pyarrow`) — write and read Parquet from Python and
  measure file sizes; the hands-on tool.
- Free reads for shape: the Parquet format overview, and the Delta Lake "what is Delta" page. Skim
  for concepts, not API.

---

#### Hands-on (all in Python — DuckDB + pyarrow)

**A. CSV vs Parquet — size and column-reading.**

    import duckdb, os
    con = duckdb.connect()
    con.execute("""
        CREATE TABLE sales AS
        SELECT i AS id, ['US','UK','IN','DE'][1+(i%4)] AS country,
               (random()*100)::DECIMAL(10,2) AS revenue
        FROM range(2_000_000) t(i)
    """)
    con.execute("COPY sales TO 'sales.csv'     (FORMAT CSV, HEADER)")
    con.execute("COPY sales TO 'sales.parquet' (FORMAT PARQUET)")
    print("CSV    MB:", round(os.path.getsize('sales.csv')/1e6, 1))
    print("Parquet MB:", round(os.path.getsize('sales.parquet')/1e6, 1))   # much smaller

1. Run it; record both sizes and the ratio. Write one sentence on *why* Parquet is smaller.
2. Query just one column from each (`SELECT SUM(revenue) FROM 'sales.csv'` vs `... 'sales.parquet'`)
   and note which has to read more of the file.

**B. Partitioning and pruning.**

    con.execute("""
        COPY (SELECT *, (id % 5) AS day FROM sales)
        TO 'sales_parts' (FORMAT PARQUET, PARTITION_BY (day))
    """)
    # look at the folder layout it created: sales_parts/day=0/, day=1/, ...
    print(con.execute("SELECT SUM(revenue) FROM 'sales_parts/*/*.parquet' WHERE day = 2").fetchone())

1. List the `sales_parts/` directory — see the `day=N/` folders. Write one sentence on how a
   `WHERE day = 2` filter avoids reading the other folders (partition pruning).
2. Now imagine partitioning by `id` instead (2M folders). Write one sentence on why that would make
   things slower, not faster (the small-files problem).

**C. Reason about formats (no code).** For each, pick CSV / Avro / Parquet / Delta and say why: (a)
a daily export another team imports into Excel; (b) the gold table your dashboard reads and that a
nightly job MERGEs into; (c) raw events streamed whole-record from Kafka; (d) a 500 GB analytical
table you scan with column filters but never update.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. In a cloud data lake, what is "a table" physically?
2. Why are CSV and JSON poor choices for storing large analytical data?
3. Explain row vs columnar layout using a `SUM(revenue)` query.
4. Name three things Parquet adds over a plain columnar dump.
5. Why does columnar data compress better than row data?
6. What is partitioning, and what is partition pruning?
7. What is the small-files / over-partitioning problem?
8. What does a transactional table format (Delta/Iceberg/Hudi) add on top of Parquet files, and why
   do you need it?
9. How does clustering differ from partitioning, and how do they stack?
10. Name the format you'd choose for: interchange, analytics-at-rest, an updatable table.

**Apply it (short scenarios — answer in 2–3 sentences):**
11. A teammate stores the 800 GB events table as gzipped CSV and queries are slow and expensive.
    What single change helps most, and why?
12. You partition a table by `customer_id` (millions of values) and it gets *slower*. Diagnose and
    propose a better partition column.
13. Two nightly jobs occasionally write the same Parquet folder and you sometimes see duplicated or
    half-written data. What's the root cause and the fix?
14. A query filters on `order_timestamp`, the table is partitioned by `order_date` and clustered by
    `order_timestamp`. Which mechanism speeds it up, partitioning or clustering?
15. You need to query "the table as it was last Tuesday" to debug a bad report. What storage feature
    makes this possible, and which format gives it to you?

**Stretch / discussion (optional):**
16. Why might you choose Iceberg over Delta (or vice-versa) for a given platform?
17. Parquet has column stats (min/max per block). Describe a `WHERE` filter that benefits hugely
    from them and one that doesn't benefit at all.

**Answer key (peek only after attempting):**
1. Files (usually Parquet) in a folder in cloud object storage. · 2. They're row-oriented, untyped
text with no compression or stats, so any query reads every byte of every row. · 3. Row stores all
fields together so you read everything; columnar stores each column separately so `SUM(revenue)`
reads only the revenue chunk. · 4. Real types, a self-describing schema, and per-block column
statistics for data skipping (plus built-in compression). · 5. A column holds one type of similar
values, which run-length/dictionary encoders crush far better than interleaved mixed-type rows. ·
6. Splitting a table's files into folders by a column's value; pruning = a filter on that column
reads only the matching folders. · 7. Partitioning on a high-cardinality column makes millions of
tiny files, and per-file overhead makes the table slower. · 8. A transaction log giving ACID
(all-or-nothing writes), time travel, and safe schema evolution; bare Parquet has no transactions,
so concurrent/failed writes corrupt it. · 9. Partitioning skips whole folders (filter on partition
key); clustering sorts within files so block stats skip blocks (filter on cluster key); use both —
coarse partition + fine cluster. · 10. CSV/JSON, Parquet, Delta (or Iceberg/Hudi). · 11. Convert to
partitioned Parquet (columnar + typed + compressed + pruning) — smaller storage and far fewer bytes
scanned. · 12. Over-partitioning created millions of tiny files; partition by date (low-cardinality,
range-filtered) instead. · 13. No transactions on bare Parquet; use a transactional format (Delta)
so writes commit atomically. · 14. Clustering — the filter is on the cluster key, not the partition
key. · 15. Time travel; a transactional format like Delta (or Iceberg/Hudi). · 16. Iceberg for huge
metadata/multi-engine neutrality; Delta for tight Databricks/Spark integration — pick for your
platform and write patterns. · 17. A range/equality filter on a sorted-ish column (e.g. timestamp)
skips many blocks; a filter on a high-cardinality column scattered randomly across all blocks skips
nothing.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Compare blob storage, a data lake, and Delta Lake.
2. What is table partitioning, and how does it relate to incremental loads?
3. Explain how clustering works in BigQuery and how it complements partitioning.
4. For a table partitioned by date and clustered by `order_timestamp`, will a filter on
   `order_timestamp` use partitioning or clustering?
5. How does Apache Iceberg optimize metadata operations?
6. How are concurrent writes handled in Delta Lake?
7. Explain Delta Lake phases: time travel, CDC, and merge.
8. You have a 200–300 GB CSV file landing in S3/SFTP. How would you load it efficiently?
9. Design a data lake holding both structured and semi-structured data.
10. How did partitioning help in a join you worked on?
11. Reconstruct the latest snapshot from a GCS landing zone partitioned by ingestion date.
12. What file formats have you used, and when would you pick Parquet vs Avro vs CSV?
13. How do you handle schema evolution when the source adds or changes a field?
14. Why might a daily incremental load to a data lake get slow for certain dates, with no errors?
15. Can you partition a BigQuery table on a VARCHAR column — and should you?

_(partition ~65, file-format/Avro/ORC ~27, parquet ~15, Iceberg/Hudi ~13, schema-evolution ~9
questions in the bank; 15 shown — more in [data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** A short DuckDB script that writes the same data to CSV and Parquet, prints both
sizes and the ratio, and writes a partitioned Parquet dataset; plus a one-paragraph note answering
"why is the gold layer stored as Delta and not CSV?" Attach your answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~3–4 hours.

---
