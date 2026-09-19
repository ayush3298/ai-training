## Advanced Track — Performance & Cost Engineering

**Goal:** Turn the scattered tuning ideas from earlier chapters into one discipline: diagnose *why*
a job is slow or expensive (read the evidence, don't guess), then apply the right lever — Spark
tuning, Delta layout, cluster sizing, or scan reduction. On cloud platforms, performance and cost are
the same problem: faster usually means cheaper.

**What we assume you know:** Chapters 3–5 and 10 — partitioning, the shuffle/skew/join material,
Delta OPTIMIZE/Z-ORDER, and warehouse cost models. This chapter is the depth sequel to Chapter 4.

**Why this matters:** "Optimize this slow/expensive pipeline" is one of the most common senior
questions in the bank (performance/cost terms appear ~190 times). The differentiator is method:
measure first, fix the actual bottleneck, verify the change.

> **Setup assumed:** conceptual; the levers were demonstrated in Chapters 4–5–10. Bring a real slow
> job (yours or the capstone's) to apply the method to.

---

#### Core concepts

**1. Measure before you tune — the bottleneck is rarely where you'd guess.**
The cardinal rule. Before changing anything, find where time/money actually goes: the **Spark UI**
(which stage is long? which task straggles? how much shuffle/spill?), the query **plan**
(`EXPLAIN`), and the warehouse's **query profile / bytes-scanned**. Tuning without evidence is how
people spend a week "optimizing" a stage that was 5% of the runtime.
- *Build consequence:* Every optimization starts with a measurement and ends with a re-measurement.
  "I think it's the join" is a hypothesis; the UI/plan is the evidence. State the before-number and
  the after-number, or you don't know if you helped.

**2. The four usual Spark bottlenecks — and the matching lever.**
From Chapter 4, in diagnostic form:
  - **Too much shuffle** → filter/aggregate earlier (shuffle fewer rows), prune columns, avoid
    needless `distinct`/`orderBy`.
  - **Skew** (one task straggles) → salting or AQE skew handling; isolate the hot/NULL key.
  - **Wrong join** (shuffling a huge side) → **broadcast** the small side.
  - **Bad partitioning** (too few = idle cores/OOM; too many tiny = overhead) → `repartition` to
    spread before heavy work, `coalesce` to consolidate before writing.
- *Build consequence:* Match the symptom in the UI to the lever: long shuffle stage, straggler task,
  big-side shuffle, or skewed partition counts each have a specific fix. **Adaptive Query Execution
  (AQE)** auto-handles several at runtime — enabling it is often the cheapest first move.

**3. Delta/lakehouse layout: the data's physical shape is half your performance.**
Read speed is mostly "how few files/bytes can the engine touch":
  - **OPTIMIZE / compaction** — fix the small-files problem so reads open fewer files.
  - **Z-ORDER / liquid clustering** — co-locate values you filter/join on so data-skipping works
    (Chapter 3/5).
  - **Partition by the range-filter column** (date), cluster by the finer filter/merge key.
  - **OPTIMIZE the MERGE key** so upserts prune to a few files instead of scanning the table.
- *Build consequence:* A table that "got slow over time" usually needs compaction + Z-ORDER on its
  hot columns, not a bigger cluster. Schedule layout maintenance as part of the pipeline — it's a
  recurring need on append/streaming tables, not a one-time setup.

**4. Cluster sizing and the cost model: right-size, autoscale, scale-to-zero.**
Compute is the bill. The levers:
  - **Right-size** — more/bigger nodes speed *parallelizable* work but do nothing for a single
    straggler (skew) — fix skew before adding machines.
  - **Autoscale** — let the cluster grow for big stages and shrink after, instead of paying for peak
    all day.
  - **Auto-suspend / scale-to-zero** — never pay for idle compute (Snowflake auto-suspend; serverless
    scale-to-zero; turn dev clusters off).
  - **Separate workloads** — isolate ETL from BI compute so one doesn't starve/slow the other.
- *Build consequence:* Adding compute is the expensive reflex; reach for it only after fixing
  skew/scan/layout, and pair it with autoscale + auto-suspend so you pay for work, not idleness.
  "Fast in dev, costly in prod" is usually scan volume + idle/oversized compute, not a need for a
  bigger box.

**5. Scan less — the universal, biggest lever.**
Across Spark, Delta, and every warehouse, the cheapest byte is the one you never read:
  - **Column pruning** — select only needed columns (huge on columnar/BigQuery per-byte billing).
  - **Partition pruning + data skipping** — filters on partition/cluster keys skip files/blocks.
  - **Predicate pushdown** — push filters into the scan (Parquet stats, Chapter 3).
  - **Pre-aggregate** — materialize repeated heavy aggregations so consumers read small results
    (Chapter 10).
- *Build consequence:* Before tuning compute, cut the data read: right columns, right partition/
  cluster keys, pre-aggregated marts. This is the first rung of the cost ladder and usually the
  biggest single win.

**6. FinOps: attribute cost, set budgets, and make the expensive things visible.**
At scale, cost is a first-class signal. Tag jobs/clusters so you can attribute spend to teams/
pipelines, watch the most expensive queries/jobs, set budgets/alerts, and review regularly. The point
isn't penny-pinching — it's catching the runaway 4-hour job or the un-pruned daily full-scan before
it's a quarterly surprise.
- *Build consequence:* Treat cost like latency — a metric you monitor and attribute, not a bill you
  discover. A dashboard of "most expensive jobs this week" turns optimization from guesswork into a
  prioritized worklist.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. A select query joining 4 tables takes ~10 seconds — how do you identify and fix it?
2. A Matillion load into Redshift takes ~4 hours — how do you optimize it?
3. The same Snowflake query is fast in dev but slow/costly in prod — what do you investigate?
4. Explain Spark optimization techniques and their use cases.
5. How do you tune read and write operations in Databricks for efficiency?
6. Explain cost-optimization strategies in Databricks.
7. How do you decide which column to index/cluster in a large warehouse, and the trade-offs?
8. A pipeline got slower over time — how did you find and fix it?
9. How did you handle a Java heap (OOM) issue from ~1 GB buffered per partition?
10. A daily incremental load slows for certain dates with no errors — how do you find the bottleneck?
11. How do you control the number of files/records written in Databricks?
12. Explain repartition vs coalesce and when each helps performance.
13. As a Databricks admin, some nodes underperform others — how do you debug?
14. How do you optimize a join where one side is 500 GB and the other 50 MB?
15. How would you reduce cost on a BigQuery workload that scans terabytes daily?

_(performance/cost-related terms appear ~190 times in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** Take one real slow job (the capstone's, or a provided one). Capture the
*before* evidence (Spark UI stage/skew or warehouse bytes-scanned), apply one targeted fix (broadcast,
Z-ORDER, partition prune, or right-size), and report the *after* number with a one-paragraph
explanation of the diagnosis → lever → result.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~3–4 hours.

---
