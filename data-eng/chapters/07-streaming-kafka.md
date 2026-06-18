## Chapter 7 — Streaming & Kafka: From Batch to Event-Driven

**Goal:** Understand event streaming — what Kafka is and why it's a *log* not a queue, how topics/
partitions/offsets/consumer-groups work, the delivery guarantees (and why "exactly-once" is hard),
and how stream processors (Spark Structured Streaming, Flink) handle late data and failures with
windows, watermarks, and checkpoints.

**What we assume you know:** Python, and Chapters 1–6 — especially batch vs streaming (Ch1),
partitioning (Ch3/4), idempotency and at-least-once delivery (Ch6).

**Why this matters:** Kafka is the backbone of real-time data, and streaming is where most
correctness bugs hide (duplicates, out-of-order events, late arrivals). Understanding the log model
and delivery semantics is what lets you build streaming pipelines that are actually correct, not
just fast.

> **Setup assumed:** conceptual + a pure-Python simulation for the hands-on (no Kafka install
> needed). Running real Kafka/Spark Structured Streaming is a stretch goal via Docker or Databricks.

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: the streaming model, Kafka as a log, topics/partitions/offsets,
  producers/consumers/groups.
- **Session 2** — concepts 5–8: replication, delivery semantics, stream processing (windows/
  watermarks/checkpoints), streaming vs batch + design + hands-on.

---

#### Core concepts

**1. Streaming is processing an unbounded, append-only log of events.**
Chapter 1 framed it as "an infinite generator." Concretely, streaming systems are built around an
**event** — an immutable record that *something happened* ("order 99 placed at 10:32"). Events are
*appended* to a never-ending log and never edited. Instead of "load yesterday's table," you "react to
each event as it arrives."
- *Build consequence:* You stop thinking in "runs" and start thinking in "a continuous flow you tap
  into." State (totals, sessions) must be maintained *incrementally* as events arrive, and you must
  handle events that arrive late or out of order — problems batch never had because batch sees the
  whole bounded dataset at once.

**2. Kafka is a distributed, durable, replayable *log* — not a queue that deletes on read.**
The instinct from app dev is a **queue**: a producer puts a message, a consumer takes it, and it's
*gone*. **Kafka is different — it's an append-only log that retains events** (for days, or forever):
  - Producers **append** events to the end.
  - Consumers **read** at their own position and reading does **not** remove anything.
  - So *many* independent consumers can read the same events, and any consumer can **replay** from
    the past (re-read from an earlier position).

Think of it less like a Python `queue.Queue` (pop removes) and more like an ever-growing list that
many readers each scan with their own bookmark.
- *Build consequence:* Because events are retained and replayable, Kafka decouples producers from
  consumers (they don't need to be up at the same time), supports many consumers off one stream
  (your warehouse loader, a fraud service, and analytics all read the same orders), and lets you
  *reprocess* history after a bug fix. This is why it's the backbone, not just a message bus.

**3. Topics, partitions, offsets: the unit of parallelism *and* of ordering.**
  - A **topic** is a named stream ("orders").
  - A topic is split into **partitions** — the log is physically divided so it can scale across
    machines and be read in parallel (like Spark partitions, Chapter 4).
  - Within a partition, every event has a monotonically increasing **offset** (its position: 0, 1,
    2…). A consumer's progress is just "I've read up to offset N in partition P."

The critical rule: **ordering is guaranteed only *within* a partition, not across the topic.** Which
partition an event goes to is chosen by its **key** (hash of the key). So if you key by
`customer_id`, all of one customer's events land in the same partition and stay ordered relative to
each other — but two different customers' events have no guaranteed order between them.
- *Build consequence:* Choose the partition **key** by what you need ordered. Need each account's
  events in order? Key by `account_id`. Pick a bad key (everything to one partition) and you lose
  parallelism (Chapter 4's skew, now in Kafka); pick no meaningful key and you lose per-entity
  ordering. More partitions = more parallel consumers, but ordering only ever holds per partition.

**4. Producers, consumers, and consumer groups: how reading scales.**
**Producers** append events. **Consumers** read them. A **consumer group** is a set of consumers
that *cooperate* to read a topic: Kafka assigns each partition to exactly one consumer in the group,
so work splits across them — add consumers (up to the partition count) to scale throughput. Each
group tracks its own **committed offset** (how far it's read) so it can resume after a restart, and
different groups are independent (your loader and your fraud service each have their own offset on
the same topic).
- *Build consequence:* Throughput scales with partitions: a group can have at most one consumer per
  partition doing useful work, so a 6-partition topic caps a group at 6 parallel consumers. Size
  partition count for your peak parallelism up front (raising it later reshuffles key→partition
  mapping). Commit offsets carefully — commit *after* successfully processing, or a crash between
  read and process can skip events.

**5. Replication and brokers: how Kafka survives a machine dying.**
A **broker** is one Kafka server; a cluster is several. Each partition is **replicated** across
brokers: one is the **leader** (handles reads/writes), the others are **followers** that copy it. If
the leader's broker dies, a follower is promoted — no data lost, no downtime. The
**replication factor** (commonly 3) is how many copies exist.
- *Build consequence:* Replication is your durability guarantee — events survive a broker failure,
  which is exactly why Kafka can be the system of record for a stream. The trade-off is that
  stronger durability (waiting for replicas to acknowledge a write, `acks=all`) costs a little
  latency; you tune that per how much you can afford to lose.

**6. Delivery semantics: at-least-once is the default, so consumers must be idempotent.**
There are three possible guarantees, and the difference is everything for correctness:
  - **At-most-once** — each event delivered 0 or 1 times: you might *lose* some. (Rare; only when
    loss is acceptable.)
  - **At-least-once** — each event delivered 1 *or more* times: you never lose, but you can get
    **duplicates** (a retry after a crash redelivers). **This is the practical default.**
  - **Exactly-once** — each event effectively processed once: no loss, no duplicates. Possible
    (Kafka transactions, idempotent producers) but adds complexity and cost.

Because at-least-once means duplicates, your consumer must be **idempotent** — exactly Chapter 6's
rule: dedup on an event id / key, or MERGE/upsert so a redelivered event doesn't double-count.
- *Build consequence:* Design every consumer assuming it *will* see the same event twice. Make
  writes idempotent (dedup-to-latest or upsert on a key) rather than reaching for full exactly-once
  semantics — it's simpler and covers most cases. "We got duplicate rows from the stream" is an
  at-least-once symptom, not a Kafka bug.

**7. Stream processing: treat the stream as a table that grows — with windows, watermarks, and checkpoints.**
Reading raw events is one thing; *computing* over them (counts per minute, sessions, joins) needs a
stream processor — **Spark Structured Streaming** (our stack) or **Apache Flink**. The key ideas:
  - **Stream-as-a-table** — Structured Streaming models the never-ending stream as a table that
    keeps gaining rows; you write almost the same DataFrame code as batch, and it runs incrementally
    on each micro-batch.
  - **Windowing** — since the stream is infinite, you aggregate over *time windows* ("orders per
    5-minute window") instead of "all rows."
  - **Watermarks** — events arrive late and out of order; a watermark says "I'll wait up to N minutes
    for stragglers, then finalize the window and drop later arrivals." It bounds how long state is
    kept. (This is a *different* watermark from Chapter 6's incremental-load high-water mark — same
    word, different idea: here it's the lateness threshold for event-time windows.)
  - **Checkpointing** — the processor periodically saves its progress (offsets + in-flight state)
    durably, so after a crash it resumes from the last checkpoint instead of reprocessing everything
    or losing state. (Flink's savepoint is a manual checkpoint for upgrades.)
- *Build consequence:* You get batch-like code over an infinite stream, but you must decide the
  window size and the watermark/lateness tolerance (too short drops real late data; too long holds
  state and delays results), and you must configure checkpointing for fault recovery. These three
  knobs are most of streaming-pipeline engineering.

**8. Streaming vs batch (again) and designing the intake: Kafka vs a database table.**
Chapter 1's rule stands: default to batch; stream only when a *named* cost of staleness justifies
the operational weight. When you do stream, a recurring design question (from the bank) is **Kafka
vs polling a database table** for ingestion:
  - **Database table as a queue** — simple, but consumers poll, it doesn't retain/replay well, and
    it loads the source DB. Fine at low volume.
  - **Kafka** — durable, replayable, multi-consumer, scales to high throughput, decouples
    producers/consumers. The right call for real-time at scale or many consumers.

A common pattern for "1000 events/min with big JSON into an ERP": producers → **Kafka** (buffer +
durability + replay) → a stream processor (validate/transform, dead-letter the invalid) → sink. Kafka
also absorbs **backpressure** — if the sink slows, events buffer in the log instead of being lost
(Flink/Spark also propagate backpressure so producers don't overwhelm consumers).
- *Build consequence:* Reach for Kafka when you need durability, replay, many consumers, or high
  throughput; a database-table queue is okay only for simple low-volume cases. Kafka in the middle
  also gives you a shock absorber (backpressure buffer) and a dead-letter path for invalid events —
  the fault-resilience the bank's design questions are really asking about.

---

#### Resources (optional — the chapter is self-contained)
- Pure-Python simulation (below) — no Kafka needed to learn the log/offset/at-least-once model.
- Stretch: **Kafka via Docker** (`docker compose` with a single broker) to publish/consume for real,
  or **Spark Structured Streaming** on Databricks Community Edition.
- Free reads for shape: the Kafka "introduction" docs (topics/partitions/consumer groups) and the
  Spark Structured Streaming "programming guide" intro.

---

#### Hands-on (pure Python — models the log; no Kafka needed)

**A. A partitioned log with offsets.** Model a topic as `partitions: list[list]`, route events by
`hash(key) % n`, and have each consumer track an offset per partition.

    import hashlib
    def part(key, n):            # which partition a key goes to
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % n

    N = 3
    topic = [[] for _ in range(N)]
    for ev in [("acct-1","a"),("acct-2","b"),("acct-1","c"),("acct-3","d"),("acct-1","e")]:
        topic[part(ev[0], N)].append(ev)     # append to the keyed partition (offset = index)

    # all acct-1 events are in ONE partition, in order:
    p = part("acct-1", N)
    print("acct-1 partition:", p, "events:", topic[p])

1. Run it; confirm every `acct-1` event is in the same partition, in arrival order. Write one sentence
   on why keying by `acct-1` guarantees its ordering but says nothing about order across accounts.

**B. At-least-once → duplicates → idempotent consumer.** Simulate a redelivery (append the same event
twice) and a consumer that processes into a dict keyed by event id. Show that processing the
duplicate again leaves the result unchanged (idempotent), versus a `list.append` consumer that
double-counts. Write one sentence linking this to "design consumers assuming duplicates."

**C. Consumer-group scaling (reason it out, no code).** A topic has 4 partitions. (a) How many
consumers in one group can do useful work in parallel? (b) What happens to the 5th consumer? (c) If a
second, separate group subscribes, does it affect the first group's offsets?

**D. Windowing & watermark sketch (no run).** In comments, sketch Structured Streaming pseudocode for
"count orders per 5-minute event-time window, tolerating up to 2 minutes of lateness." Note where the
window, the watermark, and the checkpoint location go, and what a 10-minute-late event's fate is.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. What is an event, and how is streaming's data shape different from batch's?
2. Why is Kafka called a log rather than a queue, and what does that enable?
3. What are topics, partitions, and offsets?
4. Where is ordering guaranteed in Kafka, and what controls it?
5. What does a consumer group do, and what caps its parallelism?
6. How does replication make Kafka fault-tolerant?
7. Name the three delivery semantics; which is the default and what does it force on consumers?
8. What problem do windows and watermarks solve in stream processing?
9. What does checkpointing give you?
10. When would you choose Kafka over polling a database table?

**Apply it (short scenarios — answer in 2–3 sentences):**
11. Each customer's events must be processed in order, but different customers can be parallel. How do
    you key/partition the topic, and why?
12. Your streaming sink occasionally shows duplicate rows. What's the most likely cause, and what's
    the fix that's simpler than exactly-once?
13. A 4-partition topic is your bottleneck; you add a 5th and 6th consumer to the group and nothing
    improves. Why, and what actually fixes throughput?
14. Late events (network delays) are being dropped from your per-minute counts. Which knob do you
    adjust, and what's the trade-off if you push it too far?
15. Design a fault-resilient intake for 1000 large-JSON events/min into an ERP. Name the components
    and why each is there.

**Stretch / discussion (optional):**
16. Compare Kafka and Spark Structured Streaming — what does each do, and how do they fit together?
17. Why is exactly-once expensive, and when is it actually worth it over at-least-once + idempotency?

**Answer key (peek only after attempting):**
1. An event is an immutable record that something happened; streaming data is an unbounded append-only
flow vs batch's bounded dataset. · 2. Reading doesn't delete; events are retained and replayable, so
many consumers can read and reprocess history. · 3. A topic is a named stream; partitions split it for
parallelism/ordering; an offset is an event's position within a partition. · 4. Only within a
partition; the event key (hash) decides the partition, so same-key events stay ordered. · 5. Splits a
topic's partitions among cooperating consumers; parallelism is capped at the partition count. · 6.
Each partition is replicated across brokers (leader + followers); a follower is promoted if the leader
dies, so no data is lost. · 7. At-most-once, at-least-once, exactly-once; at-least-once is default and
forces consumers to be idempotent (handle duplicates). · 8. The stream is infinite, so you aggregate
over time windows; watermarks bound how long to wait for late/out-of-order events before finalizing. ·
9. Durable saved progress (offsets + state) so a processor resumes after a crash without losing or
reprocessing everything. · 10. When you need durability, replay, many consumers, or high throughput;
a DB-table queue suits only simple low-volume cases. · 11. Key by `customer_id` so each customer's
events hash to one partition (ordered), while different customers spread across partitions (parallel).
· 12. At-least-once redelivery; make the consumer idempotent (dedup on event id / upsert) rather than
adopting full exactly-once. · 13. A group can use at most one consumer per partition (4), so the 5th/
6th idle; increase partitions to raise parallelism. · 14. Increase the watermark/allowed lateness;
too far holds state longer and delays finalizing windows. · 15. Producers → Kafka (durable buffer +
replay + backpressure) → stream processor (validate, transform, dead-letter invalid) → ERP sink; each
adds durability, decoupling, or error isolation. · 16. Kafka transports/retains the event stream;
Structured Streaming computes over it (windows/joins/aggregates) — Kafka is the pipe, the processor is
the brain. · 17. It needs transactional coordination (idempotent producer + transactional commits)
that adds latency/complexity; worth it only when duplicates are unacceptable and can't be deduped
downstream.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Explain the architecture of Kafka.
2. Explain the steps to publish a message in Kafka.
3. Compare Apache Kafka and Spark Structured Streaming.
4. Compare Kafka and a database table for data ingestion.
5. For a new source, should you use Kafka or a read replica — and why?
6. Explain subscriber flow, message broker, and event — how are they used?
7. Can there be 20,000 events in 10 minutes in Kafka? What would you consider?
8. Design a fault-resilient system for 1000 events/min with large JSON payloads into an ERP.
9. Describe your experience with Kafka Connect and the sources you worked on.
10. Describe a real-time streaming architecture (e.g. Flink + AWS, or Dataflow).
11. Explain Flink checkpoint and savepoint.
12. Explain backpressure in Flink (or a stream processor).
13. Explain key metrics to monitor in a streaming job.
14. Describe PySpark Structured Streaming with partitioning for large-scale ingestion — challenges?
15. Design a pipeline for multiple sources: batch, streaming, and real-time from Kafka.

_(Kafka ~94, real-time/event ~75, streaming ~62, broker/consumer/topic/offset ~21, Flink ~13,
checkpoint/watermark ~9 questions in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** The Python log simulation (A + B): a keyed partitioned log proving per-key ordering,
and an idempotent consumer proving a redelivered event doesn't double-count. Plus a one-paragraph
design for a fault-resilient streaming intake naming Kafka's role, the delivery semantic you assume,
and how you handle duplicates and invalid events. Attach your answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~4–5 hours.

---
