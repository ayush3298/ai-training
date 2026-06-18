## Chapter 1 — Foundations: The Data Engineering Lifecycle

**Goal:** Build a precise, working mental model of what data engineering *is* — the stages a
piece of data moves through, the concerns that ride alongside every stage, and the handful of
ideas that actually change how you design and debug a pipeline. By the end you should be able to
explain, to a friend, *why* a pipeline is shaped the way it is and what that forces you to do as
a builder.

**What we assume you know:** Python — functions, lists, dicts, loops, reading a file. **That's
it.** You do *not* need to know SQL, databases, the cloud, or what a "data warehouse" is. Every
new word is defined the first time it shows up, and new ideas are tied back to Python you already
know. If something gets a fuller treatment later, we'll say so rather than assume it.

**Why this matters:** Most outages and bad design choices in data platforms come from a wrong
mental model — treating a pipeline like a one-shot script, ignoring that it will run again and
again, or storing data for analysis the same way you'd store it for an app. Get these eight ideas
right and the rest of the course (SQL, Spark, Delta, modeling, orchestration, quality) becomes
obvious instead of mysterious.

> **Setup assumed:** none. This chapter is concepts plus a small hands-on you run in Python. The
> default stack we build on later — **Databricks + Apache Spark + Delta Lake on Azure** — gets
> stood up from Chapter 4 onward. For now, a laptop with Python is enough.

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: what data engineering is, the lifecycle, the undercurrents, and
  the two big ideas (idempotency, batch vs streaming).
- **Session 2** — concepts 5–8: where and how data is stored and transformed (ETL/ELT, the two
  kinds of database, the lakehouse, the schema tax) + the hands-on and questions.

Everything you need is taught right here — there's no book or video to go watch first.

---

### Start here: what *is* data engineering, in Python terms?

You have almost certainly already written a tiny data pipeline without calling it that. Picture a
script like this:

    rows = read_csv("sales.csv")          # 1. get the data
    clean = [r for r in rows if r.ok]     # 2. clean / reshape it
    totals = summarize(clean)             # 3. compute something useful
    write_csv("report.csv", totals)       # 4. put the result somewhere

That's the whole job in miniature: **get data from somewhere, reshape it, and put the result
somewhere useful.** Data engineering is what this becomes when the CSV is 2 billion rows instead
of 2,000, when it has to run automatically every night instead of when you press play, when five
other people depend on the output, and when "it crashed halfway" can't be allowed to corrupt the
result. Everything in this chapter is about what changes as you scale that four-line script up to
something a company runs on.

---

#### Core concepts

**1. The data engineering lifecycle is the spine: generation → ingestion → transformation → storage → serving.**
Think of the four-line script above, but named properly. Every pipeline you ever build is some
path through five stages:
  - **Generation** — data is *created* by a source: an app's database, an API, a sensor, a log
    file. (You don't usually control this — it's someone else's system.)
  - **Ingestion** — you *get the data in* (pull it on a schedule, or receive it as it's produced).
    This is `read_csv` in the toy script.
  - **Transformation** — you *reshape it*: clean it, join it, aggregate it, model it. The list
    comprehension and `summarize`.
  - **Storage** — where the data *lives* along the way. The CSVs on disk; later, a lake or a
    warehouse.
  - **Serving** — handing the finished data to whoever *uses* it: a chart, a report, a machine-
    learning model. The `write_csv`.

Storage isn't really a fifth step in a line — it's the substrate the other four sit on top of, the
way a hard drive underlies every step of your script.
- *Build consequence:* When someone hands you a vague request ("we need a sales dashboard"), your
  first move is to place it on the lifecycle: what's the *source*? how does data get *in*? what
  *transformation* makes it useful? where does it *land*? who *serves* from it? Naming the five
  stages turns a fuzzy ask into a concrete plan you can start building.

**2. The undercurrents ride alongside every stage — they're not a final step.**
Five concerns run underneath the *whole* lifecycle, not at the end of it:
  - **Security** — who is allowed to see this data?
  - **Data management** — governance, quality, and *lineage* (knowing where each number came
    from) and *metadata* (data describing your data).
  - **DataOps** — keeping it running: monitoring, alerts, and what you do at 3 a.m. when it breaks.
  - **Orchestration** — scheduling the steps in the right order (we use a tool called Airflow,
    Chapter 9).
  - **Software engineering** — the basics you already value in Python: version control, tests,
    not hard-coding secrets.

The classic beginner mistake is treating these as "stuff we'll add later."
- *Build consequence:* You think about these from the first line of code, not after the first
  incident. "Where does the password come from? How will I know if this breaks tonight? Is it safe
  to run twice?" are design questions, not cleanup chores.

**3. A pipeline runs again and again — so it must be idempotent.** *(The most important reliability idea today.)*
Your `report.csv` script runs when you press play. A real pipeline runs *on a schedule* (every
night), gets *re-run* to fix a bug, and gets *retried automatically* when it fails partway. So the
key question becomes: **what happens if this runs twice?**

You already know this distinction from Python. Compare two ways of recording a value:

    results = []
    results.append(row)     # run this twice → the row is in there TWICE

    seen = {}
    seen[key] = row         # run this twice → same result, the row is in there ONCE

`append` is *not* safe to repeat; assigning to a dict key *is*. **Idempotent** is just the fancy
word for "safe to run more than once" — running a task twice leaves you in the same state as
running it once. No duplicate rows, no revenue counted twice.

Here's the same trap when writing data out (don't worry about the exact syntax yet — read the
comments):

    # NOT idempotent: running tonight's job twice appends the same day's rows twice → duplicates
    append_rows(target, todays_rows)

    # Idempotent: replace today's slice, so a re-run lands on the same result every time
    delete_rows(target, day="2026-06-18")
    insert_rows(target, todays_rows)

- *Build consequence:* Design every task so re-running it is safe — because the scheduler *will*
  re-run it. The two workhorse patterns (you'll meet both properly later) are **upsert on a key**
  (like `seen[key] = row` — update if present, insert if not) and **partition overwrite** (replace
  a whole day's data instead of adding to it). If a task isn't safe to re-run, it's not a pipeline
  — it's a bomb that goes off on the first retry.

**4. Batch vs streaming is just "a finite list" vs "a never-ending stream."**
In Python terms: processing a **list** you can see the end of is *batch*; processing an infinite
**generator** that never stops yielding is *streaming*.
  - **Batch** — run on a schedule over a finite chunk: "every night, process yesterday's orders."
    You can see the whole input, it has a known size, and the job ends. This is most of data
    engineering.
  - **Streaming** — handle events one at a time, forever, as they happen: "the moment an order is
    placed, react to it." You never see "all" the data, only what's arrived so far.

Streaming buys you *speed* (react in seconds, not hours) and costs you *complexity* (events arrive
out of order, some arrive late, and the system must run 24/7). A simple way to choose — ask *what
does staleness cost?*

    If the user is fine with data up to an hour old        → batch. (Almost everything.)
    If one minute of old data costs money or safety        → consider streaming.
    If they "want it real-time" but can't say why          → they want batch. Build batch.

- *Build consequence:* Default to batch. Reach for streaming only when a *specific* need forces it
  (blocking fraud, live operations) — and budget for the extra always-on machinery. "Real-time" is
  a requirement to justify, never a free upgrade.

**5. ETL vs ELT — do you clean the data before or after you store it?**
Both just mean "move data from a source to where it'll be used." They differ on *when* you do the
transform (the cleaning/reshaping):
  - **ETL = Extract → Transform → Load:** clean it *first*, then store only the tidy result. Like
    filtering a list before saving it — you never keep the messy original.
  - **ELT = Extract → Load → Transform:** store the *raw* data first, then clean it afterward. Like
    saving the original file untouched, then producing tidy versions from it whenever you need.

ELT is the modern default for two practical reasons: **storage got dirt cheap** (keeping the raw
data costs pennies, so why throw it away?), and **you can re-clean later** when requirements change
— because you still have the original. ETL throws the original away; if you later need a field you
dropped, it's gone.
- *Build consequence:* You'll build **ELT**: land the raw data first (a layer we'll call
  **bronze**), then transform it forward into cleaner layers (**silver**, then **gold**). Keeping
  the raw layer is your insurance policy — when someone asks for a number you didn't compute the
  first time, you can go back to the source instead of re-collecting it.

**6. There are two opposite kinds of database — and the difference is *why this job exists*.**
First, what's a *database*? It's a separate program whose job is to store data so it survives
restarts and lets many people read and write it safely at once — more than a Python dict (which
vanishes when your program ends) and more than a file (which gets corrupted if two programs write
at once). There are two families, tuned for opposite jobs:
  - **OLTP — the app's database (transactional).** Handles many tiny reads and writes: "create
    *this one* order," "look up *this one* user." It's what the website talks to. (Postgres, MySQL.)
  - **OLAP — the analytics database (analytical).** Handles a few *huge* questions: "total revenue
    across *all 50 million* orders, grouped by country." This is the warehouse. (Snowflake,
    BigQuery, Databricks.)

If you run a giant analytics question directly against the app's database, you slow the whole
website down for real users — the classic rookie mistake. So we *copy* data out of the OLTP
database into an OLAP one built for big questions. **That copy is literally the ingestion half of
your job.**

Why the analytics database is built differently — with hand-checkable numbers. Analytics databases
store data *by column* instead of *by row*. Say a table has 50 columns and is 100 GB total, and
your question only needs 2 columns (`country`, `revenue`):

    Stored by row:     to read 2 columns you still sweep past all 50  → ~100 GB read
    Stored by column:  you read only the 2 columns you asked for       → ~4 GB read   (2 of 50)

Same question, ~25× less data read. That's the whole reason analytics databases are "columnar."
- *Build consequence:* Never point a dashboard at the live app database. Use a row-by-row store
  for the app, a column store for analytics — and recognize that the gap between those two worlds
  is exactly the work you're being hired to do: get data from one into the other.

**7. Storage and compute are separated now — and that's what a "lakehouse" is built on.**
Two words, plainly: **storage** = where the bytes sit (a hard disk, or cloud "object storage" like
a giant shared folder). **Compute** = the processor doing work on those bytes. Old systems welded
them together: to store more data you had to rent a bigger always-on machine, and you paid for it
even when nobody was asking questions. The modern setup **separates** them: data sits in cheap
cloud storage, and you spin up processing power *only when you need it* and turn it off after. Three
places analytical data can live:
  - **Data lake** — a giant cheap folder of raw files. Flexible, but it has no rules: nothing stops
    bad or duplicate data, so it can rot into a "data swamp."
  - **Data warehouse** — a strict, fast, organized database. Reliable, but historically pricier and
    awkward for messy/unstructured data.
  - **Lakehouse** — the cheap-folder storage of a lake *plus* a rules layer on top (called Delta,
    Iceberg, or Hudi) that adds reliability and structure. You get the warehouse's guarantees at the
    lake's price. This is the home of our default stack.
- *Build consequence:* "Where does the data live?" has three answers with real trade-offs. Because
  storage and compute are separate, you can keep *all* your raw data cheaply and only pay for
  processing while it runs — and it's why we build on a lakehouse (Chapter 5) rather than an
  old-style warehouse.

**8. "Schema" is the agreed shape of your data — and you always pay to enforce it, the only question is when.**
A **schema** is just the expected shape: which fields exist and what type each is — like a Python
`dataclass` or "every row is a dict with keys `id` (int), `name` (str), `price` (float)." Real data
breaks its shape constantly: a price arrives as `"12.00"` (text) instead of `12.0` (number), a
field goes missing, a new one appears. Someone has to deal with that. You have two moments to do it:
  - **Schema-on-write** — check and fix the shape *as you store it*. Strict and safe (bad rows get
    caught immediately), but rigid (the source changes and your loader rejects everything).
  - **Schema-on-read** — store whatever arrives, sort out the shape *when you read it later*.
    Flexible (nothing gets rejected), but you've only delayed the pain — a field quietly changes
    type and every report downstream breaks at once.

There is no "no schema" option. There's only *when you pay.*
- *Build consequence:* The bronze→silver→gold layering from concept 5 is the deliberate answer:
  **bronze** uses schema-on-read (take in everything raw, never lose data), and **silver/gold** use
  schema-on-write (enforce the shape before anyone builds reports on it). You choose to pay the
  schema tax at a clear boundary — not by accident, at 2 a.m., when a dashboard fills with errors.
  This sets up data modeling, which is Chapter 2.

---

#### Resources (optional — the chapter is self-contained)
- **DuckDB** (free; `pip install duckdb`, or run it in the browser at shell.duckdb.org) — a tiny
  analytics database you drive *from Python*. It's the playground for the hands-on below; no account
  or server needed.
- The free docs pages for the modern stack's vocabulary, skimmed for shape not sales pitch:
  Databricks' "What is a lakehouse?" and dbt's "Analytics Engineering" overview.
- These are optional — you can finish the whole chapter with just Python installed.

---

#### Hands-on (all in Python — `pip install duckdb` and go)

You'll touch SQL here (the language for asking questions of tables). You don't need to know it yet —
each snippet gives you the exact query string to pass; Chapter 2 teaches SQL properly.

**A. Place a real pipeline on the lifecycle (no code).**
1. Pick any data thing you've seen — a sales dashboard, a "people who bought this" feature, a daily
   report email. Write one line per stage: source → ingestion → transformation → storage → serving.
2. For each of the five undercurrents, write one sentence: what would security / data management /
   DataOps / orchestration / software engineering mean for *this* pipeline?

**B. Feel why analytics needs a different database.**

    import duckdb
    con = duckdb.connect()
    # make ~2 million fake sales rows
    con.execute("""
        CREATE TABLE sales AS
        SELECT i AS id,
               ['US','UK','IN','DE'][1 + (i % 4)] AS country,
               (random() * 100)::DECIMAL(10,2) AS revenue
        FROM range(2_000_000) t(i)
    """)
    # a big analytical question (touches few columns, all rows)
    print(con.execute("SELECT country, SUM(revenue) FROM sales GROUP BY country").fetchall())
    # a tiny transactional question (one row)
    print(con.execute("SELECT * FROM sales WHERE id = 12345").fetchall())

Run it. Then write two sentences: which question is "OLAP-shaped" and which is "OLTP-shaped," and
why running the big one against a live app's database would hurt real users.

**C. Make a non-idempotent step, then fix it.**

    con.execute("CREATE TABLE report AS SELECT * FROM sales WHERE country = 'US'")
    before = con.execute("SELECT COUNT(*) FROM report").fetchone()[0]
    # the NON-idempotent move: append the same slice again
    con.execute("INSERT INTO report SELECT * FROM sales WHERE country = 'US'")
    after = con.execute("SELECT COUNT(*) FROM report").fetchone()[0]
    print(before, after)   # after is DOUBLE — re-running corrupted the result

Now make it idempotent: before inserting, delete the `country='US'` rows first
(`DELETE FROM report WHERE country='US'`), then insert. Run *that* twice and confirm the count
stays the same. Write one sentence on why a scheduler that auto-retries makes this non-optional.

**D. Batch vs streaming — name the cost of staleness (no code).**
For three cases — a nightly finance report, a fraud check at checkout, a "trending now" widget —
decide batch or streaming, and write the *cost of one hour of stale data* for each in a sentence.
Notice it's that cost, not the word "real-time," that decides.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. Name the five stages of the data engineering lifecycle, in order.
2. What are the five undercurrents, and why are they not a "final step"?
3. In one line, what does *idempotent* mean, and which Python operation is a good analogy for it?
4. What's the core difference between batch and streaming, in "list vs generator" terms?
5. What's the difference between ETL and ELT, and why is keeping the raw data useful?
6. What's the difference between an OLTP and an OLAP database, and why do we copy data between them?
7. Why does an analytics database store data by column instead of by row?
8. What does a lakehouse add on top of a plain data lake?
9. What is a "schema," and what does schema-on-read vs schema-on-write decide?
10. What does it mean that storage and compute are "separated," and why does that save money?

**Apply it (short scenarios — answer in 2–3 sentences):**
11. A teammate says, "just point the new dashboard at the app's live Postgres database, it's
    simpler." What goes wrong, and what do you suggest instead?
12. Your nightly job has auto-retry turned on. Last night it failed halfway, retried, and now one
    day's revenue is counted twice. What's the root cause, and how do you fix it?
13. A manager insists a report be "real-time." How do you turn that into a real decision, and what's
    your default if they can't name a cost of staleness?
14. Six months ago your loader dropped a field nobody needed. Now someone needs it. Why does an ELT
    design save you, and what would a strict ETL design have cost you?
15. You're loading data from an API whose shape changes without warning. Where do you stay
    schema-on-read, where do you enforce a schema, and why?

**Stretch / discussion (optional):**
16. We called storage "the substrate," not a step in a line. What goes wrong in your mental model if
    you picture storage as just one box between "transform" and "serve"?
17. Give one realistic case where the right answer is genuinely streaming, and say exactly which
    cost of staleness justifies the extra always-on machinery.

**Answer key (peek only after attempting):**
1. Generation → ingestion → transformation → storage → serving. · 2. Security, data management,
DataOps, orchestration, software engineering; they run under *every* stage, so you design them in
from the start. · 3. "Safe to run more than once" — like `d[key] = value` (re-running changes
nothing), unlike `list.append` (re-running adds a duplicate). · 4. Batch = a finite list you can
see the end of and finish; streaming = a never-ending generator you process forever as items
arrive. · 5. ETL cleans before storing (keeps only the tidy result); ELT stores raw first then
cleans — keeping raw lets you re-clean when needs change. · 6. OLTP handles many tiny app
reads/writes; OLAP handles a few huge analytical scans; we copy OLTP→OLAP so big questions don't
slow the app. · 7. Analytical questions touch few columns over many rows, so reading just those
columns moves far less data (and compresses better). · 8. A reliability/structure layer (ACID,
schema enforcement, time travel) on top of cheap lake storage — warehouse guarantees at lake
prices. · 9. A schema is the expected fields and types; schema-on-read vs -on-write decides
*when* you enforce that shape (at query time vs at load time). · 10. Bytes sit in cheap storage
while processing power is spun up only when needed and turned off after — so you don't pay for an
idle machine. · 11. Big analytics scans slow the app for real users; copy the data into an
analytics (OLAP) store and serve the dashboard from there. · 12. The step appended instead of
replacing (not idempotent); switch to upsert-on-key or replace-that-day so retries are safe. ·
13. Ask what one hour (or minute) of stale data costs; if they can't name a cost, build batch. ·
14. ELT kept the raw data, so you re-derive the field from it; strict ETL discarded it at load, so
it's gone unless you can re-ingest from the source. · 15. Bronze stays schema-on-read (accept
everything raw, lose nothing); enforce the schema entering silver so gold and all reports build on
a stable shape. · 16. Storage decides cost, speed, and guarantees for every other stage; treating
it as one box hides choices (row vs column, lake vs warehouse, how long to keep data) that actually
decide whether the pipeline works. · 17. e.g. blocking fraud at checkout — a stale decision lets a
bad transaction through, so seconds of delay have direct money cost that justifies always-on
streaming.

---

#### Interview drill (self-test)

These are real questions from the data-engineering interview bank, scoped to this chapter. As a
fresher you won't have polished answers to all of them yet — that's fine. Try each in a sentence
now, then come back and re-answer as the course fills in the gaps. They tell you where you're
headed.

1. What is a data warehouse, and how does it differ from a data lake?
2. Can we use both a data lake and a data warehouse in the same project? When would you?
3. Compare blob storage, a data lake, and Delta Lake.
4. Explain the bronze–silver–gold (medallion) layers for a data lake.
5. Describe batch processing. When is it the right choice?
6. Explain the difference between streaming inserts and batch loads.
7. What is the difference between OLTP and OLAP systems?
8. Explain ETL vs ELT, and why ELT is common on modern cloud stacks.
9. Describe, stage by stage, an end-to-end data pipeline you would build.
10. What are the types of data sources, and how do structured, semi-structured, and unstructured
    data differ?
11. What does a data engineer do, and how is the role different from a data analyst or scientist?
12. Sketch a pipeline that pulls data from an app's database into a warehouse and feeds a dashboard.
13. How would you check data quality in a pipeline, and what would you check for?
14. An app table has 2 billion+ rows; the team wants it in a data lake plus a 3-month slice in
    Postgres. How would you move it?
15. What are SLAs and SLOs for a data pipeline, and how would you measure reliability?

_(466 foundation-relevant questions matched in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** A short note — 6–8 bullets — titled *"What a data platform is, and what that
means for building one."* Each bullet = one concept + its practical implication, in your own words.
Attach your answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~2–3 hours (reading + hands-on + questions).

---
