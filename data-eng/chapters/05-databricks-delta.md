## Chapter 5 — Databricks & Delta Lake: The Default Platform Up Close

**Goal:** Learn the platform the rest of this course builds on — **Databricks** (managed Spark +
notebooks + governance) and **Delta Lake** (the transactional table format) — well enough to build
a medallion pipeline, update tables idempotently with MERGE, recover with time travel, and keep
tables fast with OPTIMIZE. This is the third heavy chapter and the heart of the default stack.

**What we assume you know:** Chapters 1–4 — the lifecycle, the medallion idea, Parquet, the
transaction-log concept, and Spark (driver/executor, lazy DataFrames, shuffle). Databricks *is*
managed Spark, so Chapter 4 is the engine underneath everything here.

**Why this matters:** Databricks/Delta is the most-asked topic in the entire interview bank, and for
good reason: it's where ingestion, transformation, storage, and serving come together on one
platform. The patterns here — MERGE, medallion, time travel, OPTIMIZE/Z-ORDER, Unity Catalog — are
the daily tools of the job.

> **Setup assumed:** **Databricks Community Edition** (free) for the full experience with Spark SQL.
> To run Delta locally with no cluster, the hands-on uses the pure-Python `deltalake` package
> (`pip install deltalake pandas pyarrow`) — same on-disk format, same concepts, no Spark needed.

**Suggested split across three working sessions (heavy chapter):**
- **Session 1** — concepts 1–3: the platform, Delta = Parquet + log, time travel.
- **Session 2** — concepts 4–6: MERGE/upsert, schema enforcement/evolution, the medallion.
- **Session 3** — concepts 7–8 (+ note): OPTIMIZE/Z-ORDER/VACUUM, Unity Catalog, DLT/Auto Loader +
  hands-on.

---

#### Core concepts

**1. Databricks is managed Spark + notebooks + governance — the lakehouse as a product.**
Running Spark yourself means provisioning machines, installing Spark, wiring up storage and
security — a lot of plumbing. **Databricks** is that, managed for you: you get notebooks to write
code, clusters that spin up on demand (and scale to zero when idle), Delta Lake built in, and a
governance layer (Unity Catalog). You write PySpark or SQL in a notebook; Databricks runs it on a
cluster against data in your cloud storage.
  - The **lakehouse** (Chapter 1) is the *idea*; Databricks is one *implementation* of it.
  - It runs on all three clouds; our default is **Azure** (as "Azure Databricks").
- *Build consequence:* You focus on the data logic, not the cluster plumbing. But the bill is driven
  by cluster *uptime and size*, so "turn it off when idle" and "size it right" become real cost
  levers (concept 7 and the Advanced track). Managed doesn't mean free of engineering — it moves
  your engineering from infra to data.

**2. Delta Lake = Parquet data files + a transaction log — and the log is the whole trick.**
A Delta table on disk is just a folder: your **Parquet** data files (Chapter 3) plus a
`_delta_log/` subfolder of JSON commit files. Each commit records, atomically, which files were
*added* and *removed* in that version. To read the table "now," an engine reads the log to learn the
current set of files — not whatever happens to be in the folder.

That one indirection — *the log, not the folder listing, defines the table* — is what gives Delta
its powers:
  - **ACID transactions** — a write appends one commit; until that commit lands, readers see the old
    version. No half-written or duplicated reads, even with concurrent writers.
  - The log is an ordered **history of versions**, which directly enables time travel (concept 3).
- *Build consequence:* Because a commit is all-or-nothing, you can safely have concurrent jobs and
  failed-and-retried jobs without corrupting the table — the exact reliability bare Parquet lacked
  (Chapter 3). This is why every silver/gold table you build is Delta, not raw Parquet.

**3. Time travel: query or restore the table as of an older version, because the log kept history.**
Since the log records every version, you can read the past:

    -- Spark SQL on Databricks
    SELECT * FROM sales VERSION AS OF 12;                 -- the table at commit 12
    SELECT * FROM sales TIMESTAMP AS OF '2026-06-17';     -- as of a point in time
    RESTORE TABLE sales TO VERSION AS OF 12;              -- roll a bad write back

Uses: debugging ("what did the report see yesterday?"), recovering from a bad load (restore), and
reproducibility (pin an ML training run to a table version).
- *Build consequence:* A bad batch that corrupted a table is no longer a disaster — you `RESTORE` to
  the prior version instead of rebuilding from source. (Caveat: time travel only reaches back as far
  as old files still exist — `VACUUM`, concept 7, deletes them past a retention window, which is the
  trade-off between recoverability and storage cost.)

**4. MERGE (upsert) is the workhorse: update-or-insert in one atomic, idempotent statement.**
Chapter 1 said tasks must be idempotent; MERGE is *the* tool that makes table updates idempotent.
"Upsert" = update the row if its key exists, insert it if not. One statement:

    MERGE INTO customers AS t
    USING updates       AS s   ON t.id = s.id
    WHEN MATCHED     THEN UPDATE SET *
    WHEN NOT MATCHED THEN INSERT *;

Re-run the same MERGE with the same source and the table ends in the same state — no duplicates.
This is how you apply a batch of changes, build SCD Type 2 dimensions (Chapter 2), and apply CDC
change-streams (Chapter 6). The bank's question "does MERGE do a full scan or partial?" — Delta uses
the log's file statistics to touch only the files that *could* contain matching keys (a partial scan
when the data is laid out well; partition/Z-ORDER on the merge key makes this much cheaper —
concept 7).
- *Build consequence:* Reach for MERGE whenever you apply updates/changes to a table — it replaces
  fragile delete-then-insert logic with one atomic, re-runnable operation. Lay the table out so the
  merge key prunes files, or MERGE on a huge table reads everything.

**5. Schema enforcement by default; schema evolution on request.**
Delta is **schema-on-write** for silver/gold (Chapter 1's schema tax, paid here): by default it
*rejects* a write whose columns/types don't match the table — so a malformed upstream batch fails
loudly instead of silently corrupting downstream reports. When you genuinely want to add a column,
you opt in to **schema evolution** (`mergeSchema`) so the change is deliberate, not accidental.
- *Build consequence:* You get a guardrail for free: bad-shaped data is stopped at the door rather
  than poisoning gold. When a source legitimately adds a field, you evolve the schema on purpose.
  This is the enforcement that makes "schema-on-write at the silver boundary" real.

**6. Medallion architecture: bronze → silver → gold — the standard way to organize a lakehouse.**
This is the most-asked pattern in the bank, and it's just ELT (Chapter 1) with named, purposeful
layers:
  - **Bronze (raw)** — land source data *as-is*, append-only, schema-on-read. Your durable,
    replayable copy of history. If anything downstream is wrong, you can rebuild from bronze.
  - **Silver (cleaned/conformed)** — deduplicated, typed, validated, joined into clean entities.
    Schema enforced here. This is the "single source of truth" most pipelines build from.
  - **Gold (business-level)** — aggregated, modeled for consumption: the star schemas (Chapter 2)
    and aggregates your dashboards and ML read directly.

Each layer is Delta; each step (bronze→silver, silver→gold) is a transformation you can re-run
idempotently (MERGE / partition-overwrite).
- *Build consequence:* When asked to "design a pipeline," default to medallion: land raw to bronze
  (never lose data), clean/validate into silver (one source of truth), aggregate into gold (what
  users touch). It gives you replayability (rebuild from bronze), a clear place for quality checks
  (entering silver), and a clean serving layer (gold).

**7. Keep tables fast: OPTIMIZE (compact), Z-ORDER (cluster), VACUUM (clean up) — and why each matters.**
Streaming/incremental writes create many small files (Chapter 3's small-files problem); Delta gives
you the maintenance tools:
  - **OPTIMIZE** — compacts many small files into fewer big ones, so reads open fewer files.
  - **Z-ORDER (by col)** — co-locates rows with similar values of a column into the same files, so
    the log's min/max stats let queries *skip* files (data-skipping, Chapter 3's clustering). Z-ORDER
    on the columns you filter/join on most (often the merge key).
  - **VACUUM** — physically deletes data files no longer referenced by the log and older than a
    retention window (default 7 days). Reclaims storage — but **removes the ability to time-travel
    past that window**, so it's a deliberate trade-off.
  - (Newer Databricks: **liquid clustering** and **predictive optimization** automate much of this.)
- *Build consequence:* A table that gets slower over time (a real bank symptom) usually needs
  OPTIMIZE + Z-ORDER on its filter/join keys. Schedule maintenance as part of the pipeline; and set
  VACUUM retention with eyes open — aggressive VACUUM saves storage but shortens how far back you can
  recover.

**8. Unity Catalog: one governance layer — who can see what, and where each number came from.**
**Unity Catalog** is Databricks' governance layer across workspaces. It gives you a three-level name
`catalog.schema.table` (e.g. `prod.sales.fact_orders`), centralized **access control** (grant
table/column/row access to users and groups), **lineage** (it tracks which tables/notebooks produced
which — answering "where did this number come from?"), and an audit trail.
- *Build consequence:* This is how you answer the security and data-management *undercurrents* from
  Chapter 1 in practice: permissions, lineage, and audit live in one place instead of being bolted
  on per-table. When asked "how do you handle governance on fact/dimension tables," the answer is
  Unity Catalog grants + lineage, not ad-hoc file permissions.

> **Two names you'll hear (and meet again in Chapter 6):** **Auto Loader** incrementally and
> idempotently ingests new files as they land in cloud storage (it tracks what it's already seen) —
> the standard way to feed bronze. **Delta Live Tables (DLT)** is a *declarative* way to define
> medallion pipelines: you declare each table and its quality expectations, and Databricks manages
> the dependencies, retries, and incremental updates. Both are conveniences on top of the concepts
> above — know what they're for; you don't need them to build a medallion pipeline by hand.

---

#### Resources (optional — the chapter is self-contained)
- **Databricks Community Edition** (free) — notebooks + Spark SQL; the real way to feel the platform
  and read the Spark UI.
- **`deltalake`** Python package (`pip install deltalake pandas pyarrow`) — write/read/time-travel Delta
  tables locally with no Spark; the hands-on tool below.
- Free reads for shape: the Delta Lake docs "Delta Lake quickstart" and the Databricks "Medallion
  architecture" page.

---

#### Hands-on (Python — `pip install deltalake pandas pyarrow`; or Spark SQL on Community Edition)

**A. Build a Delta table and look at the log.**

    import pandas as pd
    from deltalake import write_deltalake, DeltaTable

    df = pd.DataFrame({"id": [1, 2, 3], "city": ["London", "Tokyo", "Lima"]})
    write_deltalake("dim_customer", df)            # creates Parquet + _delta_log/
    dt = DeltaTable("dim_customer")
    print("version:", dt.version())                # 0
    print(dt.to_pandas())

1. Look inside the `dim_customer/` folder — note the Parquet file(s) **and** the `_delta_log/`
   folder of JSON commits. Write one sentence: what does the log define that a plain folder listing
   doesn't?

**B. Time travel.**

    write_deltalake("dim_customer", pd.DataFrame({"id":[1],"city":["Paris"]}),
                    mode="overwrite")              # version 1
    print(DeltaTable("dim_customer", version=0).to_pandas())   # the OLD data is still readable

Confirm version 0 still returns London/Tokyo/Lima. Write one sentence on how this saves you after a
bad overwrite.

**C. MERGE / upsert is idempotent.** Using `DeltaTable(...).merge(...)` (delta-rs) or Spark SQL
`MERGE INTO`, upsert a change for `id=1` (city → "Berlin") and a new `id=4`. Run the *same* merge
twice. Confirm: id=1 updated once (not duplicated), id=4 inserted once. Write one sentence linking
this to Chapter 1's idempotency rule.

**D. Reason about a medallion design (no code).** For a sales pipeline, write 2–3 lines each for
bronze, silver, gold: what lands there, what transformation happens, and where you'd (a) enforce
schema, (b) put a data-quality check, (c) Z-ORDER and on which column.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. What does Databricks give you on top of plain Spark?
2. What two things sit in a Delta table's folder, and which one defines the table?
3. How does the transaction log give Delta ACID guarantees?
4. What is time travel and name two uses.
5. What does MERGE do, and why is it the idempotent way to apply changes?
6. What does schema enforcement protect you from, and how do you intentionally evolve a schema?
7. Describe the three medallion layers and the purpose of each.
8. What do OPTIMIZE, Z-ORDER, and VACUUM each do?
9. What's the trade-off when you run VACUUM aggressively?
10. What does Unity Catalog provide (name three things)?

**Apply it (short scenarios — answer in 2–3 sentences):**
11. A nightly job sometimes fails halfway and you've seen duplicated rows in the target. How do
    Delta + MERGE eliminate this class of bug?
12. A bad batch overwrote your gold table with garbage an hour ago. What's the fastest recovery, and
    what feature enables it?
13. A Delta table has gotten slow; the folder has hundreds of thousands of tiny files and queries
    filter on `event_date`. What two maintenance commands do you run and why?
14. An upstream team starts sending an extra column. Your write now fails. What's happening, and what
    are your two options?
15. Asked to "design the pipeline for a sales dashboard on Databricks," sketch it in medallion terms
    and say where schema is enforced and where quality checks live.

**Stretch / discussion (optional):**
16. "Does MERGE scan the whole table?" — explain what determines whether it's a full or partial
    scan, and how you'd make it partial.
17. When would Delta Live Tables be worth adopting over hand-written medallion notebooks, and when
    would it be overkill?

**Answer key (peek only after attempting):**
1. Managed clusters (on-demand, scale-to-zero), notebooks, built-in Delta, and governance (Unity
Catalog) — no infra plumbing. · 2. Parquet data files and a `_delta_log/`; the log defines which
files are the table now. · 3. Each write is one atomic commit in the log; until it lands readers see
the prior version, so writes are all-or-nothing and concurrency-safe. · 4. Querying/restoring the
table as of an older version or timestamp; uses: debugging and recovery/rollback (and reproducible
ML). · 5. Update-if-key-exists, insert-if-not, atomically; re-running with the same source yields the
same state, so no duplicates on retry. · 6. It rejects writes whose schema doesn't match (stops bad
data); opt into `mergeSchema` to add a column deliberately. · 7. Bronze = raw append-only replayable
landing; silver = cleaned/typed/deduped source of truth (schema enforced); gold = aggregated/modeled
for consumption. · 8. OPTIMIZE compacts small files; Z-ORDER co-locates similar values for data
skipping; VACUUM deletes unreferenced old files to reclaim storage. · 9. It shortens how far back you
can time-travel/recover, in exchange for less storage. · 10. Three-level naming, centralized
access control, lineage (and audit). · 11. Delta commits are atomic (no half-writes) and MERGE is
idempotent (re-running doesn't duplicate), so a failed-and-retried load lands the same correct
state. · 12. RESTORE the table to the previous version — enabled by time travel/the transaction log.
· 13. OPTIMIZE (compact the tiny files) and Z-ORDER by `event_date` (so filters skip files). · 14.
Schema enforcement rejected the new column; either evolve the schema (`mergeSchema`) on purpose or
fix/align the source. · 15. Bronze: raw sales events; silver: cleaned/typed/deduped orders (enforce
schema + quality checks entering here); gold: star schema/aggregates for the dashboard. · 16. Delta
uses file min/max stats from the log to skip files that can't match; partitioning/Z-ORDER on the
merge key makes it a partial scan, otherwise it reads everything. · 17. DLT is worth it for many
interdependent tables needing managed orchestration/quality/incremental logic; overkill for a
simple, stable handful of tables.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Explain how Delta Lake works in Databricks.
2. Explain Medallion architecture and why to use it.
3. Explain Delta Lake phases: time travel, CDC, and merge.
4. Can MERGE be used in Databricks? Does `MERGE INTO ... ON T.id=S.id` do a full or partial scan?
5. Describe the gold layer: structure, purpose, and implementation.
6. How are concurrent writes handled in Delta Lake?
7. Explain Databricks architecture end to end.
8. Explain cost optimization strategies in Databricks.
9. Can you control the number of files and records written in Databricks? How?
10. Explain Databricks Auto Loader briefly.
11. How do you handle governance on fact and dimension tables (Unity Catalog)?
12. A pipeline got slower over time — how did you find and fix it?
13. Best practices for deploying views in Databricks across dev/prod (no dev-specific schema)?
14. Describe a Teradata→Databricks (or Oracle→Databricks) migration: steps, stack, considerations.
15. How would you build a report after deleting records in a Delta table?

_(Databricks ~229, optimize/Z-ORDER/vacuum ~107, medallion ~61, delta table ~54, merge/upsert ~29,
unity catalog ~21, time travel ~11 questions in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** A working Delta exercise (local `deltalake` or Community Edition): create a table,
overwrite it, read an older version via time travel, and apply the same MERGE twice proving no
duplicates. Plus a one-page medallion design for a sales pipeline marking where schema is enforced,
where quality checks live, and which column you'd Z-ORDER. Attach your answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~5–7 hours across the three sessions.

---
