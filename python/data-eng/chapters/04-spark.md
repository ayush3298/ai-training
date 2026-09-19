## Chapter 4 — Spark & Distributed Processing: How the Work Actually Runs

**Goal:** Understand what Apache Spark is, how it splits a job across many machines, and the
handful of mechanics (lazy plans, the shuffle, skew, join strategies, partitioning) that decide
whether your job runs in 2 minutes or 2 hours. This is the second of the three heavy chapters and
the engine the whole default stack runs on.

**What we assume you know:** Python (ideally a little pandas — a DataFrame is a table in memory),
and Chapters 1–3 (partitioning, Parquet, OLAP).

**Why this matters:** Spark is the processing engine behind Databricks and most large-scale data
work. Almost every "my job is slow / runs out of memory" problem is one of four things — a shuffle,
skew, the wrong join, or bad partitioning — and you can't fix what you can't see. This chapter
gives you the mental model to read a slow job and know which lever to pull.

> **Setup assumed:** conceptual. To run code, the easiest free option is **Databricks Community
> Edition** (a hosted notebook, no install) or local PySpark (`pip install pyspark`, needs Java).
> You do *not* need a cluster to learn the model — the same code runs on one machine or a thousand.

**Suggested split across three working sessions (heavy chapter):**
- **Session 1** — concepts 1–3: why Spark, the architecture, lazy DataFrames.
- **Session 2** — concepts 4–6: narrow vs wide, the shuffle, skew.
- **Session 3** — concepts 7–8: join strategies, partitioning & caching + hands-on.

---

#### Core concepts

**1. Spark exists for when data doesn't fit (or finish) on one machine.**
You know the wall: a pandas DataFrame lives in one machine's RAM, so a 500 GB file on a 16 GB laptop
just won't load, and a single CPU chewing through a billion rows takes forever. **Spark** is a
framework that spreads both the *data* and the *work* across a **cluster** — many machines
cooperating as one. You write what looks like pandas; Spark splits the data into chunks, ships a
copy of your code to each machine, and runs them in parallel.

The key new idea is **distributed**: there's no single place that holds all the data or does all the
work. That one fact causes everything hard (and everything fast) about Spark.
- *Build consequence:* Reach for Spark when data is too big for one machine's memory or too slow for
  one CPU. For data that fits comfortably on one box, plain Python/pandas/DuckDB is simpler and
  often *faster* — distribution has overhead. "Use Spark for everything" is an anti-pattern; use it
  when the size justifies it.

**2. The architecture: one driver coordinates many executors; data lives in partitions.**
A Spark job has three pieces:
  - **Driver** — the brain. It runs your program, builds the plan, and hands out work. Think of it
    as your `main()` that coordinates but doesn't crunch the data itself.
  - **Executors** — the workers. Each is a process on a cluster machine with its own CPU and memory;
    each does a slice of the work on a slice of the data.
  - **Cluster manager** — assigns executors to your job (YARN, Kubernetes, or Databricks' own).

The data itself is split into **partitions** — chunks (often ~128 MB) that are the *unit of
parallelism*. One executor core processes one partition at a time. 200 partitions and 50 cores →
50 run at once, in 4 waves.
- *Build consequence:* Parallelism is capped by partition count and core count. Too *few* partitions
  → idle cores and giant per-task memory (out-of-memory risk); too *many* tiny partitions → overhead
  dominates. Most "Spark is slow" or "OOM" issues are really "the data is partitioned wrong for this
  cluster."

**3. DataFrames are lazy: transformations build a plan, only an action runs it.**
This trips up everyone coming from pandas, where each line executes immediately. In Spark, operations
split in two:
  - **Transformations** (`select`, `filter`, `withColumn`, `groupBy`, `join`) are **lazy** — they
    don't run; they just add a step to a *plan* (a DAG, directed acyclic graph of steps). Like
    building up a recipe without cooking.
  - **Actions** (`show`, `count`, `collect`, `write`) **trigger** the whole plan to actually run.

        df2 = df.filter(df.country == "US")   # nothing happens yet — just planning
        df3 = df2.groupBy("city").count()      # still nothing
        df3.show()                             # NOW Spark runs the whole chain

Why bother? Because Spark sees the *whole* plan before running and optimizes it — the **Catalyst
optimizer** reorders and prunes (e.g. pushes your filter down to read fewer rows, drops columns you
never use). You writing "filter last" can still execute "filter first."
- *Build consequence:* Don't expect print-debugging line by line — nothing runs until an action.
  And don't call actions (like `count()`) casually in the middle of a pipeline: each one re-runs the
  plan from the start unless you cached (concept 8). Laziness is why Spark can be fast, and why its
  execution feels surprising at first.

**4. Narrow vs wide transformations: the difference is whether data has to move between machines.**
This is the single most important performance distinction in Spark.
  - **Narrow** transformation — each output partition depends on *one* input partition. `filter`,
    `select`, `withColumn`: every executor works on its own chunk alone, no coordination. Cheap and
    parallel.
  - **Wide** transformation — an output partition depends on *many* input partitions, so data must
    be moved across the network so related rows meet. `groupBy`, `join`, `distinct`, `orderBy`:
    to group by city, every "London" row scattered across 200 partitions must be brought together.

That movement is called a **shuffle** (concept 5) and it's the expensive part of almost every slow
job.
- *Build consequence:* When optimizing, scan your code for the wide operations — `groupBy`, `join`,
  `distinct`, `orderBy`, `repartition`. Those are where the time and the failures live. Narrow ops
  are basically free; wide ops are what you design around.

**5. The shuffle is the costly heart of Spark — and it's what splits a job into stages.**
A **shuffle** moves data across the network so rows with the same key land on the same executor
(needed for any group/join). It's expensive because it hits the network *and* disk: every executor
writes its data out sorted by key, then every executor reads back the pieces meant for it (an
all-to-all exchange). This is why a `groupBy` over a huge table can dwarf the rest of the job.

The vocabulary the interviews ask about falls straight out of this:
  - A **job** is everything one action triggers.
  - It's split into **stages** at each shuffle boundary — a stage is a run of narrow ops that need
    no data movement; a new stage begins after each shuffle.
  - Each stage runs as **tasks** — one task per partition, run on the executors in parallel.

So "job → stages → tasks" is just: the action, cut into pieces at every shuffle, each piece run once
per chunk of data.
- *Build consequence:* Minimize shuffles and the data they move. Filter and aggregate *early* (fewer
  rows to shuffle), avoid needless `distinct`/`orderBy`, and prefer joins that skip the shuffle
  (concept 7). When you read the Spark UI, the long stages are almost always the shuffle stages —
  that's where you look first.

**6. Data skew: when one key has most of the rows, one task does all the work while the rest idle.**
The shuffle assumes keys spread out evenly. **Skew** is when they don't — e.g. you `groupBy
customer_id` and one customer (or a `NULL`) owns 90% of the rows. After the shuffle, all those rows
land on *one* executor, so 199 tasks finish in seconds and 1 task ("the straggler") runs for an
hour — and may run out of memory and crash the whole job. Same total work, catastrophically
unbalanced.

The fixes (the bank asks these directly):
  - **Salting** — break the hot key into many by appending a random suffix (`customer_id + "_" +
    rand(0..N)`), aggregate the pieces, then combine. Spreads the hot key across N tasks.
  - **Adaptive Query Execution (AQE)** — modern Spark detects skew at runtime and splits the skewed
    partition automatically; often just turning it on is enough.
  - **Filter out / handle the junk key** (the giant `NULL` bucket) separately.
- *Build consequence:* A job where "one task takes forever and the rest are done" is the textbook
  skew signature — don't add more machines (the idle ones won't help the straggler); fix the
  *distribution* with salting/AQE. Recognizing skew from the symptom is a core debugging skill.

**7. Join strategies: broadcast the small table to avoid shuffling the big one.**
Joining two tables normally shuffles *both* so matching keys meet — a **shuffle (sort-merge) join**,
fine when both are large. But the bank's classic — *500 GB joined to 50 MB* — has a much better
answer: a **broadcast join.** If one side is small enough, Spark sends a full copy of it to *every*
executor, so each executor joins its chunk of the big table against the in-memory small table —
**no shuffle of the 500 GB at all.**

        from pyspark.sql.functions import broadcast
        big.join(broadcast(small), "key")     # ships `small` to every executor; big table stays put

Spark auto-broadcasts tables under a size threshold (~10 MB by default, tunable); for the 50 MB
table you'd raise the threshold or hint `broadcast()` explicitly.
- *Build consequence:* When a join is slow and one side is small (a dimension table, a lookup),
  broadcast it — you trade a tiny bit of memory on each executor for eliminating the expensive
  shuffle of the huge side. Choosing the join strategy by the *sizes of the two sides* is a
  first-line tuning move.

**8. Partitioning and caching: the two everyday tuning levers — `repartition` vs `coalesce`, and `cache`.**
  - **`repartition(n)`** — reshuffle the data into `n` evenly-sized partitions. A *full shuffle*
    (expensive), but it can both *increase* parallelism and *fix* imbalance. Use before a heavy wide
    stage when partitions are too few or skewed.
  - **`coalesce(n)`** — *merge* existing partitions down to `n` *without* a full shuffle (cheap), but
    only *decreases* count and can leave them uneven. Use to combine many tiny output partitions
    into a few big files before writing (fixing the small-files problem from Chapter 3).
  - **`cache()` / `persist()`** — keep a DataFrame in executor memory after it's first computed.
    Because Spark is lazy and re-runs the plan on every action, if you reuse a DataFrame several
    times (e.g. an expensive cleaned dataset feeding three reports), cache it once so the expensive
    work happens *once*, not three times.
- *Build consequence:* `repartition` to *spread out* before heavy work (accept the shuffle to gain
  parallelism/balance); `coalesce` to *consolidate* before writing (avoid a shuffle, fix tiny
  files); `cache` only a DataFrame you genuinely reuse (caching something used once just wastes
  memory). These three cover most hand-tuning you'll do before reaching for cluster sizing.

---

#### Resources (optional — the chapter is self-contained)
- **Databricks Community Edition** (free hosted notebooks) — the no-install way to run the PySpark
  below and *see the Spark UI* (jobs/stages/tasks). The UI is the real teacher here.
- Local **PySpark** (`pip install pyspark`, needs Java 8/11/17) if you prefer your own machine.
- For the model: the "Spark: The Definitive Guide" architecture chapter, or any "Spark jobs/stages/
  tasks" explainer — read for the mental picture, not the API.

---

#### Hands-on (PySpark — Community Edition or local)

> The point is to *see* the execution model in the Spark UI, not just get answers. After each
> action, open the UI and look at the stages and tasks.

**A. Lazy vs action.**

    df = spark.range(0, 10_000_000).withColumnRenamed("id", "n")
    f = df.filter(df.n % 2 == 0)          # transformation — nothing runs (no job in the UI yet)
    print(f.count())                       # action — now a job appears

Confirm no job appears until `count()`. Write one sentence on why laziness lets Spark optimize.

**B. See a shuffle create a new stage.**

    df = spark.range(0, 10_000_000).withColumn("k", (col("id") % 100))
    df.groupBy("k").count().show()         # groupBy = wide = shuffle → two stages in the UI

In the UI, find the stage boundary at the shuffle. Write one sentence mapping "job → stages →
tasks" onto what you see.

**C. Make skew, then a broadcast join.**
1. Build a DataFrame where one key value owns ~90% of rows (`when(id < 9_000_000, 0).otherwise(id)`),
   `groupBy` it, and watch one task run far longer than the others in the UI — that's skew.
2. Create a tiny lookup DataFrame and join with `broadcast(small)`; confirm in the UI that the big
   side is **not** shuffled. Write one sentence on when broadcasting is the right call.

**D. repartition vs coalesce (no cluster needed to reason).** In comments, answer: which would you
use to (a) go from 4 to 200 partitions before a big join, (b) combine 5,000 tiny output files into
20 before writing? Explain the shuffle difference.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. When should you use Spark instead of pandas/DuckDB — and when not?
2. What are the roles of the driver, the executors, and a partition?
3. What's the difference between a transformation and an action? Give two of each.
4. Why is Spark lazy, and what does that buy you?
5. Narrow vs wide transformation — what's the real difference?
6. What is a shuffle and why is it expensive?
7. Explain "job → stage → task" and what determines a stage boundary.
8. What is data skew, what's its symptom in the UI, and name one fix.
9. When is a broadcast join the right choice, and what does it avoid?
10. `repartition` vs `coalesce` — what does each do and when do you use it?
11. When does `cache()` help, and when is it a waste?

**Apply it (short scenarios — answer in 2–3 sentences):**
12. A PySpark job runs fine on small data but OOMs on a full day. One task in the last stage runs
    10× longer than the rest. Diagnose and give a fix.
13. You join a 500 GB fact table to a 50 MB lookup and it's shuffling forever. What do you change?
14. A job calls `.count()` for logging at five points in the pipeline and is mysteriously slow.
    What's happening, and what's the fix?
15. Your output is 10,000 tiny Parquet files and downstream reads are slow. Which Spark operation
    fixes it before the write, and why that one (not the other)?

**Stretch / discussion (optional):**
16. You add 50 more executors to a skewed job and it's no faster. Explain why, in terms of where the
    work actually is.
17. Adaptive Query Execution can fix some skew automatically. Why can't it fix *everything*, and
    when would you still salt manually?

**Answer key (peek only after attempting):**
1. Use Spark when data exceeds one machine's memory or one CPU's time; for data that fits, pandas/
DuckDB is simpler and often faster. · 2. Driver coordinates and builds the plan; executors do the
work on their slices; a partition is a data chunk and the unit of parallelism. · 3. Transformations
(filter, select, groupBy, join) build a lazy plan; actions (count, show, collect, write) trigger
execution. · 4. It can see the whole plan and optimize (push filters down, prune columns) before
running anything. · 5. Narrow: each output partition comes from one input partition, no data
movement; wide: output needs many input partitions, forcing a shuffle. · 6. Moving data across the
network (and through disk) so same-key rows meet; it's an all-to-all exchange hitting network + I/O.
· 7. A job is one action's work, split into stages at each shuffle, each stage run as one task per
partition; a shuffle starts a new stage. · 8. One key holds most rows so one task gets most of the
data and straggles (and may OOM); fix with salting, AQE, or isolating the hot/NULL key. · 9. When
one side is small enough to copy to every executor; it avoids shuffling the large side. · 10.
repartition does a full shuffle to n balanced partitions (increase/rebalance); coalesce merges down
without a full shuffle (decrease, e.g. fewer output files). · 11. Helps when a DataFrame is reused
across multiple actions; wasteful when used once. · 12. Skew — one heavy key lands on one task; salt
the key or enable AQE skew handling (and/or repartition), don't just add machines. · 13. Broadcast
the 50 MB side (`broadcast(small)` or raise the threshold) so the 500 GB table isn't shuffled. · 14.
Each `count()` is an action that re-runs the whole lazy plan; remove them or cache the reused
DataFrame. · 15. `coalesce(n)` — merges tiny partitions into fewer files without a full shuffle;
`repartition` would also work but pays an unnecessary shuffle. · 16. The work is concentrated on one
task/partition (skew); idle extra executors can't take a slice of a single task — you must
redistribute the data. · 17. AQE splits skewed shuffle partitions it detects at runtime, but can't
help skew it can't see or non-shuffle hotspots; salt manually when a known hot key dominates a join/
group.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Explain Spark architecture: job, stage, task, and execution.
2. How is the Spark DAG built internally?
3. Explain the difference between narrow and wide transformations.
4. Explain the Catalyst optimizer in PySpark.
5. Are RDDs mutable? How do RDDs relate to DataFrames?
6. Explain repartition and coalesce — when to use which?
7. Explain watermarking and salting for data skew. How do you do it in PySpark?
8. A join is between a 500 GB and a 50 MB table — how do you improve it? Write the code.
9. Describe a scenario where the join strategy in PySpark impacted job efficiency and how you chose.
10. Compare Pandas and Spark for data processing.
11. How do you debug and optimize PySpark code for both performance and accuracy?
12. How did you handle a Java heap (OOM) issue from ~1 GB of buffered records per partition?
13. Translate this SQL to PySpark: a JOIN + GROUP BY with AVG, HAVING COUNT > 2, ORDER BY.
14. Given login/logout events, generate session id, start, end, and duration in PySpark.
15. A daily incremental PySpark load slows down for certain dates with no errors — how do you find
    and fix the bottleneck?

_(executor/driver/stage ~19, narrow/wide/lazy/catalyst ~18, repartition/coalesce ~9, skew/salting
~6, broadcast ~5, cache/persist ~5 questions in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** Run hands-on A–C on Community Edition (or local), and capture **one Spark-UI
screenshot** showing a job split into two stages at a shuffle, plus a short note: "where was the
shuffle, and what would I change if one task straggled?" Attach your answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~5–7 hours across the three sessions.

---
