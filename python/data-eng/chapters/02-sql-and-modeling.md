## Chapter 2 — SQL & Data Modeling

**Goal:** Learn the language you'll use every single day (SQL) and the two ways to *shape* data —
**normalized** (for apps) and **dimensional** (for analytics) — so you can both query data and
design where it lives. This is one of the three biggest chapters in the course; take your time.

**What we assume you know:** Python, and Chapter 1 (the lifecycle, OLTP vs OLAP, the schema idea).
You do **not** need any prior SQL. We build it from the first `SELECT`.

**Why this matters:** SQL is the universal language of data — every database, warehouse, Spark, and
dbt speaks it. And *modeling* (how you arrange data into tables) decides whether a query is one
clean line or an unmaintainable mess. Most slow dashboards and "wrong number" bugs trace back to a
modeling choice, not a code bug.

> **Setup assumed:** Python with DuckDB (`pip install duckdb`) — a real SQL database you drive from
> Python, no server needed. Everything here runs on your laptop.

**Suggested split across three working sessions (this is a heavy chapter):**
- **Session 1** — concepts 1–4: what SQL is, the shape of a query, how tables relate, and joins.
- **Session 2** — concepts 5–6: aggregation & window functions, then normalization.
- **Session 3** — concepts 7–9: dimensional modeling, slowly changing dimensions, and why queries
  get slow (indexes) + the hands-on.

---

#### Core concepts

**1. SQL is *declarative*: you describe the result you want, not the steps to get it.**
In Python you say *how*: loop, filter, accumulate. In SQL you say *what* the answer looks like and
the database figures out how to compute it. Compare — same task, "US customers' names":

    # Python: you spell out the steps
    names = []
    for c in customers:
        if c["country"] == "US":
            names.append(c["name"])

    -- SQL: you describe the result; the engine plans the steps
    SELECT name FROM customers WHERE country = 'US';

A **table** is just a list of rows where every row has the same named fields — picture a
`list[dict]` where all the dicts share keys. A **query** is a question you ask of one or more
tables; it returns a new table.
- *Build consequence:* Because you only declare *what*, the database is free to choose *how* —
  which is what lets it run the same query over 10 rows or 10 billion. It also means performance is
  the engine's job to plan and yours to *enable* (concept 9), not something you hand-code.

**2. Every query is the same six clauses — and they don't run in the order you write them.**
You write `SELECT` first, but the database runs it almost last. The clauses:

    SELECT   country, COUNT(*) AS n     -- 5. pick/compute the columns to return
    FROM     customers                  -- 1. which table
    WHERE    signup_year = 2026         -- 2. keep only rows matching a condition (before grouping)
    GROUP BY country                    -- 3. collapse rows into one row per group
    HAVING   COUNT(*) > 100             -- 4. keep only groups matching a condition (after grouping)
    ORDER BY n DESC;                     -- 6. sort the result

Logical run order is **FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY**. The one that trips
everyone up: `WHERE` filters *individual rows before grouping*; `HAVING` filters *groups after*.
"Customers who signed up in 2026" is a `WHERE`; "countries with more than 100 of them" is a
`HAVING`.
- *Build consequence:* Knowing the order tells you *where* to put a filter. Filtering early (in
  `WHERE`) means fewer rows to group and sort — faster and cheaper. Putting a row-level condition in
  `HAVING` by mistake works but makes the engine do far more work than it needs to.

**3. Tables relate through keys — a primary key identifies a row, a foreign key points to one.**
You wouldn't store a customer's full address on every one of their 500 orders — you'd store the
customer once and have each order *reference* them. In Python you'd key a dict by id:

    customers = {1: {"name": "Mara", ...}}      # keyed by a unique id
    order = {"id": 99, "customer_id": 1, ...}   # points back to a customer by its id

In SQL those ids have names:
  - **Primary key** — the column that *uniquely* identifies a row (a customer's `id`). No duplicates,
    never null.
  - **Foreign key** — a column in another table that *points to* a primary key elsewhere (an order's
    `customer_id`). It's how two tables connect.
- *Build consequence:* Keys are the seams along which you split and re-join data. Get them right and
  every join is clean; get them wrong (a non-unique "key," a missing foreign key) and you get
  duplicated or orphaned rows that quietly corrupt every total downstream.

**4. A join combines two tables on a key — and the join *type* decides what happens to non-matches.**
This is the single most important SQL skill. A join stitches rows from table A to rows in table B
wherever a key matches — like merging two `list[dict]`s on a shared id. The four you need:
  - **INNER JOIN** — keep only rows that match in *both* tables. (Customers *who have* orders.)
  - **LEFT JOIN** — keep *all* left rows; fill nulls where the right has no match. (*All* customers,
    with their orders if any — this is the one you reach for most.)
  - **RIGHT JOIN** — the mirror image (all right rows). Rarely needed; flip the tables and use LEFT.
  - **FULL OUTER JOIN** — keep everything from both sides, nulls where either is missing.

The trap that bites everyone — **fan-out**: a join can *multiply* rows. If one customer has 3
orders, joining customers→orders produces 3 rows for that customer, and a naive
`SUM(customer.credit_limit)` now triple-counts. Worked example with the bank's own numbers — table A
has 5 rows, table B has 4, with 2 matching keys:

    INNER JOIN  → 2 rows      (only the matches)
    LEFT JOIN   → 5 rows      (all of A; non-matches get NULLs from B)
    FULL OUTER  → 7 rows      (2 matched + 3 unmatched A + 2 unmatched B)
    CROSS JOIN  → 20 rows     (5 × 4 — every row paired with every row; no key at all)

- *Build consequence:* Pick the join type by asking "what should happen to rows with no match?" —
  dropping them (INNER) is a silent data-loss bug if you wanted LEFT. And whenever you aggregate
  *after* a join, check for fan-out: are you summing a value that got duplicated by the join? This
  is the #1 cause of "the dashboard number is too high."

**5. `GROUP BY` collapses rows into summaries; a window function summarizes *without* collapsing.**
`GROUP BY` turns many rows into one-per-group: `SELECT country, SUM(revenue) ... GROUP BY country`
gives you one row per country and you *lose* the individual rows. Often you want a summary *attached
to every original row* — a running total, a rank, "each order alongside its customer's total." That's
a **window function**: it computes across a "window" of related rows but keeps every row.

    -- rank orders within each customer, newest first, and keep all rows
    SELECT
      order_id, customer_id, order_time,
      ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_time DESC) AS rn
    FROM orders;

`OVER (PARTITION BY customer_id ORDER BY order_time DESC)` reads: "for each customer, ordered newest
first." Two patterns from the interview bank you'll use constantly:
  - **Latest record per group** (deduplication / "current state from a change log): the query above,
    then `WHERE rn = 1` keeps only each customer's newest row.
  - **Nth highest** (e.g. 3rd-highest salary): `DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk`,
    then `WHERE rnk = 3`.

(`ROW_NUMBER` = unique 1,2,3 even on ties; `RANK` = 1,2,2,4 on a tie; `DENSE_RANK` = 1,2,2,3.)
- *Build consequence:* "Latest row per key" is the everyday tool for turning a stream of changes
  into current state (you'll use it for CDC in Chapter 6 and Delta in Chapter 5). Reaching for a
  self-join or a Python loop instead is slower and far more error-prone.

**6. Normalization: split data so each fact lives in exactly one place — this is how *app* databases are shaped.**
"Normalizing" means organizing tables so you don't repeat data. If you stored the customer's name on
every order, then a name change means updating 500 rows — miss one and your data now *disagrees with
itself* (an "update anomaly"). The fix: store each customer once in a `customers` table; orders just
reference the `customer_id`. The rules of thumb (you rarely cite them by number, but know the idea):
  - **1NF** — one value per cell (no comma-separated lists stuffed in a column).
  - **2NF / 3NF** — every column depends on the table's key and nothing else; anything that's really
    about a *different* thing gets its own table.

The goal: **every fact in one place, so there's one place to update it.** This is ideal for OLTP
(the app), where writes are constant and correctness is everything.
- *Build consequence:* When you design the *source* side or a clean intermediate layer, normalize —
  it keeps writes safe and data consistent. But normalization means *more tables and more joins to
  answer a question*, which is exactly the wrong trade-off for analytics — which is why analytics
  uses the opposite shape (concept 7).

**7. Dimensional modeling: deliberately *de*normalize into facts and dimensions — this is how *analytics* is shaped.**
For analytics you optimize for *reading* big questions fast and for humans understanding the model —
so you accept some duplication. The **Kimball star schema** splits the world into two kinds of table:
  - **Fact table** — the measurable events, one row per event, with numbers you add up (a `sales`
    fact: `quantity`, `revenue`) plus foreign keys to its dimensions. Long and skinny; millions of
    rows.
  - **Dimension tables** — the descriptive "who / what / where / when" you slice by (`customer`,
    `product`, `store`, `date`). Wide and short; they hold the text attributes.

Drawn out, the facts sit in the middle pointing at dimensions around them — a *star*:

        dim_date  ─┐
        dim_store ─┼─►  fact_sales  ◄─┬─ dim_product
        dim_customer ─┘                └─ (measures: quantity, revenue)

"Total revenue by product category and month" becomes `fact_sales` joined to `dim_product` and
`dim_date` — a couple of clean joins, fast, and readable. (A *snowflake schema* further normalizes
the dimensions into sub-tables; usually not worth the extra joins — prefer a flat star.)
- *Build consequence:* This is the target shape for your **gold** layer (Chapter 1's medallion) and
  what your dashboards read from. When asked to "design the data model for a sales dashboard," you
  draw a star: identify the *event* (the fact) and the things you slice it *by* (the dimensions).

**8. Slowly Changing Dimensions (SCD): how you handle a dimension attribute that changes over time.**
A customer moves from London to Paris. What do you do to their `dim_customer` row? Two common
answers:
  - **SCD Type 1 — overwrite.** Just update the city to Paris. Simple, but you've *destroyed
    history* — every past order now looks like it was from Paris. Fine when history doesn't matter
    (fixing a typo).
  - **SCD Type 2 — keep history.** Don't overwrite; *add a new row* for the Paris version and mark
    the old one as expired, using validity columns:

        customer_id  city    valid_from   valid_to     is_current
        42           London   2024-01-01   2026-06-18   false       ← old version, closed off
        42           Paris    2026-06-18   9999-12-31   true        ← current version

    Now a sale from 2025 still joins to the London row (correct history), and today's joins to
    Paris. The "latest record per key" window function from concept 5 is exactly how you find the
    current row.
- *Build consequence:* The question "do we need to preserve history for this attribute?" is a
  *design decision you must ask explicitly* for every dimension. Choose Type 1 and stakeholders lose
  the ability to report "as it was back then"; choose Type 2 and your dimension grows and every join
  must filter to the right version. Getting this wrong is expensive to fix after the data's loaded.

**9. Why a query gets slow — and the index, the one tuning tool you'll reach for first.**
A database with no help must read *every row* to find the ones you want — a "full table scan," like
`for row in ten_million_rows`. An **index** is a pre-sorted lookup structure on a column, exactly
like the index at the back of a book: instead of reading all 500 pages to find "joins," you check the
index and jump straight to page 212.

    -- without an index, this scans the whole table; with one on customer_id, it jumps straight there
    SELECT * FROM orders WHERE customer_id = 42;
    CREATE INDEX idx_orders_customer ON orders(customer_id);

Indexes aren't free: they speed up *reads* that filter/join on that column but slow down *writes*
(every insert must also update the index) and use space. The first things to check when a query is
slow: (a) are you filtering/joining on an indexed column? (b) are you reading more columns than you
need (`SELECT *` vs the 3 columns you use)? (c) is a join fanning out and producing far more rows than
expected? The tool that shows you what the engine is actually doing is `EXPLAIN` (put it before any
query) — you'll meet it properly when we tune Spark in the Advanced track.
- *Build consequence:* When "a 4-table join takes 10 seconds" (a real bank question), you don't
  guess — you read the plan, check the join keys are indexed, filter earlier, and select fewer
  columns. In analytics stores the same idea appears as **partitioning** and **clustering** (Chapters
  3 and 5): physically arranging data so the engine can skip most of it.

---

#### Resources (optional — the chapter is self-contained)
- **DuckDB** (`pip install duckdb`) — drives real SQL from Python; the hands-on tool.
- **sqlbolt.com** and **pgexercises.com** — free, in-browser, click-through SQL practice. The single
  best way to drill joins and window functions until they're reflex.
- For modeling: Kimball's "star schema" and "SCD" concepts are widely summarized online — read for
  the shapes, not the book.

---

#### Hands-on (all in Python — DuckDB)

**A. Build two related tables and join them.**

    import duckdb
    con = duckdb.connect()
    con.execute("CREATE TABLE customers(id INT, name TEXT, country TEXT)")
    con.execute("INSERT INTO customers VALUES (1,'Mara','US'),(2,'Ito','JP'),(3,'Lena','US')")
    con.execute("CREATE TABLE orders(id INT, customer_id INT, amount DECIMAL(8,2))")
    con.execute("INSERT INTO orders VALUES (10,1,50),(11,1,20),(12,2,99)")  # note: customer 3 has none

    print(con.execute("""
        SELECT c.name, COUNT(o.id) AS n_orders, COALESCE(SUM(o.amount),0) AS spent
        FROM customers c
        LEFT JOIN orders o ON o.customer_id = c.id
        GROUP BY c.name ORDER BY spent DESC
    """).fetchall())

1. Run it. Lena has no orders — confirm she still appears (that's what LEFT JOIN buys you). Change
   `LEFT` to `INNER` and watch her vanish. Write one sentence on when that disappearance is a bug.

**B. See fan-out bite.** Add a second orders row for customer 1, then
`SELECT c.name, SUM(c.id) FROM customers c JOIN orders o ON o.customer_id=c.id GROUP BY c.name`.
Notice `SUM(c.id)` for Mara is inflated because the join duplicated her row. Write one sentence
explaining what went wrong and how you'd avoid it.

**C. Window function — latest order per customer.**

    print(con.execute("""
        SELECT id, customer_id, amount FROM (
          SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY id DESC) AS rn
          FROM orders
        ) WHERE rn = 1
    """).fetchall())

Then change it to find the **2nd** highest `amount` overall using `DENSE_RANK()`. Write one sentence
on why this is better than sorting and grabbing a row by position.

**D. Build a tiny star schema and do an SCD Type 2 update (no exact syntax needed — reason it out).**
1. Sketch (on paper or in comments) a star for sales: one `fact_sales` (with `customer_id`,
   `product_id`, `date_id`, `quantity`, `revenue`) and three dimensions.
2. In `dim_customer`, give customer 42 a `city`, `valid_from`, `valid_to`, `is_current`. Now they
   move cities: write the two steps (close off the old row by setting its `valid_to` and
   `is_current=false`; insert a new current row). Confirm a `WHERE is_current` query returns only the
   new city, and that the old row is still there for history.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. What does it mean that SQL is "declarative," and how is that different from a Python loop?
2. Put the six query clauses in *logical execution* order. Why is it not the written order?
3. What's the difference between `WHERE` and `HAVING`?
4. What's the difference between a primary key and a foreign key?
5. INNER vs LEFT JOIN — what happens to rows with no match in each?
6. What is join "fan-out," and what bug does it cause?
7. How does a window function differ from `GROUP BY`?
8. Why do app databases normalize while analytics databases denormalize?
9. In a star schema, what goes in a fact table vs a dimension table?
10. SCD Type 1 vs Type 2 — what does each do when an attribute changes?
11. What is an index, and what's the trade-off for having one?

**Apply it (short scenarios — answer in 2–3 sentences):**
12. A revenue dashboard shows numbers that are too high. You're summing an amount after joining
    customers to their many orders. What's likely happening and how do you check?
13. You need "each employee's 3rd-highest monthly sales." Which tool — GROUP BY or a window function
    — and roughly how do you write it?
14. Stakeholders ask, "what was each customer's *region at the time of each past order*?" Your
    `dim_customer` uses Type 1 (overwrite). Why can't you answer this, and what would have prevented
    the problem?
15. A 4-table join takes 10 seconds. Walk through the first three things you'd check, in order.

**Stretch / discussion (optional):**
16. When would a snowflake schema (normalized dimensions) actually be worth the extra joins over a
    flat star?
17. You're turning a table of every status-change event into "current status per order." Which single
    window-function pattern does this, and why is it better than a self-join?

**Answer key (peek only after attempting):**
1. You describe the result you want and the engine plans the steps; a loop spells out the steps
yourself. · 2. FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY; SELECT is written first for
readability but can't run until the rows to return exist. · 3. WHERE filters individual rows before
grouping; HAVING filters groups after aggregation. · 4. A primary key uniquely identifies a row in
its table; a foreign key in another table points to that primary key. · 5. INNER keeps only matches;
LEFT keeps all left rows and fills NULLs where the right has no match. · 6. A join multiplying rows
when one row matches many; aggregates over the result then double/triple-count. · 7. GROUP BY
collapses rows to one per group; a window function computes across related rows but keeps every row.
· 8. Apps normalize so each fact has one place to update (safe writes, consistency); analytics
denormalizes so big read queries need fewer joins and are easier to understand. · 9. Fact = the
measurable events with additive numbers + foreign keys; dimensions = the descriptive attributes you
slice by. · 10. Type 1 overwrites and loses history; Type 2 adds a new versioned row and expires the
old one, preserving history. · 11. A pre-sorted lookup on a column that speeds filtered reads/joins
but slows writes and costs space. · 12. Fan-out: the join duplicated each customer across their
orders, inflating the SUM; check row counts before/after the join, or aggregate orders first then
join. · 13. Window function: `DENSE_RANK() OVER (PARTITION BY employee ORDER BY monthly_sales DESC)`
then filter to rank = 3. · 14. Type 1 overwrote the old region, so past orders now point at the
current one; Type 2 (versioned rows with valid_from/valid_to) would have preserved the region as it
was. · 15. (a) Are the join keys indexed? (b) Are you selecting more columns/rows than needed? (c) Is
a join fanning out? — then read EXPLAIN. · 16. When a dimension is huge and its sub-attributes are
reused across many dimensions, or storage/consistency of the dimension genuinely matters more than
query simplicity. · 17. `ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY event_time DESC)` then
`WHERE rn = 1`; one pass, no expensive self-join, and it's clear what "latest" means.

---

#### Interview drill (self-test)

Real questions from the data-engineering bank, scoped to this chapter. Try each now; you'll answer
them cleanly by the end of the course.

1. Explain the order of execution of a SQL query with SELECT, WHERE, and GROUP BY.
2. Given table A (5 rows) and table B (4 rows, 2 matching): how many rows from a full outer join? a
   cross join?
3. Explain this query: `SELECT salary FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC)
   AS rnk FROM employee) t WHERE rnk = 3;`
4. Explain a query using `ROW_NUMBER() OVER (PARTITION BY id ORDER BY timestamp DESC)` with `WHERE
   rn = 1` — what does it pick?
5. Explain the difference between a star schema and a snowflake schema.
6. Explain slowly changing dimensions. Walk through implementing SCD Type 2.
7. Explain denormalization with a customer-sales example.
8. A SELECT joining 4 tables takes ~10 seconds. How do you identify and fix the problematic join?
9. How do you decide which column to index in a large warehouse? What are the trade-offs?
10. Explain `SELECT business_key, COUNT(*) FROM t GROUP BY business_key HAVING COUNT(*) > 1;` — what
    is it for?
11. How do you delete duplicate rows where id and name match but address differs?
12. Design a star schema for sales (region, customer, product, sales, sales_item) and write an
    analytical query against it.
13. Explain a CROSS JOIN with an example (5 rows × 4 rows).
14. Explain a query that uses `SUM(CASE WHEN status='Pending' THEN 1 ELSE 0 END)` — what's the
    technique called?
15. How do you handle fact and dimension tables and keep them updated?

_(Joins ~37, window/rank ~30, star/dimension ~29, normalization ~14, SCD ~11, optimization ~13
questions in the bank; 15 shown — more in [data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** Two things. (1) A working DuckDB script that builds `customers`/`orders`, does a
LEFT JOIN aggregate, and a "latest-row-per-customer" window query. (2) A one-page diagram of a sales
**star schema** with a note on which dimension you'd make SCD Type 2 and why. Attach your answers to
questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~5–7 hours across the three sessions (this is a heavy chapter).

---
