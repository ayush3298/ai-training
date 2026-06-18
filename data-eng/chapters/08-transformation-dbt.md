## Chapter 8 — Transformation & dbt: ELT You Can Test and Trust

**Goal:** Learn dbt — the standard tool for the "T" in ELT — and the software-engineering discipline
it brings to transformation: models and a dependency graph from `ref()`, materializations,
sources/staging/marts layering, tests, Jinja/macros, and how dbt fits with Airflow and CI/CD.

**What we assume you know:** SQL (Chapter 2), the medallion layers (Chapters 1/5), and Python
(Jinja templating will feel familiar). No prior dbt.

**Why this matters:** Raw SQL transformations sprawl into hundreds of un-versioned, untested queries
with hidden dependencies — the classic "data spaghetti." dbt turns transformation into real software:
version-controlled, tested, documented, with an automatic build order. It's one of the most-asked
tools in the bank.

> **Setup assumed:** Python with **dbt + DuckDB adapter** (`pip install dbt-duckdb`) — run a real dbt
> project on your laptop, no warehouse needed. The same project runs against Databricks/Snowflake/
> BigQuery by swapping the adapter.

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: what dbt is, models & `ref()`, materializations, sources/staging/
  marts.
- **Session 2** — concepts 5–8: tests, Jinja/macros, docs/lineage, dbt in the workflow + hands-on.

---

#### Core concepts

**1. dbt does the "T" in ELT: you write `SELECT`s, dbt manages everything around them.**
After Extract+Load (Chapters 5–6) your raw data sits in the warehouse/lakehouse. **dbt (data build
tool)** transforms it *in place* using SQL. The shift in thinking: you don't write `CREATE TABLE ...
INSERT ...` and orchestration glue — you write a `SELECT` that defines what a table *should contain*,
and dbt handles the `CREATE`/`INSERT`/`DROP`, the build order, and the boilerplate.

    -- models/customer_orders.sql — you write ONLY this SELECT
    SELECT c.id, c.name, COUNT(o.id) AS n_orders, SUM(o.amount) AS spent
    FROM {{ ref('stg_customers') }} c
    LEFT JOIN {{ ref('stg_orders') }} o ON o.customer_id = c.id
    GROUP BY 1, 2
    -- dbt turns this into a table or view named customer_orders, in dependency order

dbt = SQL **plus software engineering**: version control (it's just files in git), tests,
documentation, and dependency management.
- *Build consequence:* Transformation becomes code you review, test, and version — not ad-hoc
  queries someone ran once. You stop writing DDL and orchestration boilerplate and focus on the
  business logic in the `SELECT`. This is the discipline that keeps a warehouse maintainable past a
  dozen tables.

**2. A model is one `SELECT`; `ref()` builds the dependency graph so dbt knows the build order.**
Each **model** is a `.sql` file containing one `SELECT` → one table or view. Models depend on each
other through `{{ ref('other_model') }}` instead of hard-coded table names. From those `ref()`s dbt
builds a **DAG** (the dependency graph, Chapter 4's idea) and figures out the correct build order
automatically — build `stg_customers` before `customer_orders` because the latter `ref`s the former.
- *Build consequence:* You never manually sequence transformations again — `ref()` *is* the
  dependency, and `dbt run` builds everything in the right order, in parallel where possible. It also
  means `ref()` (not raw table names) is mandatory: it's what makes the graph correct and lets dbt
  swap schemas between dev and prod (Chapter 5's "no dev-specific schema in prod" problem, solved).

**3. Materializations: choose whether a model becomes a view, a table, or an incremental build.**
The same `SELECT` can be physically built different ways — the **materialization**:
  - **view** — dbt creates a view; recomputed on every read. Cheap to build, no storage; good for
    light/early models.
  - **table** — dbt rebuilds the full table each run. Fast to read, costs a full recompute each time.
  - **incremental** — only process *new/changed* rows each run and merge them in, instead of
    rebuilding everything (Chapter 6's incremental idea, in dbt). Essential for big event tables —
    you'd never rebuild a billion-row fact nightly.
  - **ephemeral** — not built at all; inlined into models that ref it (like a CTE).
- *Build consequence:* Match materialization to size and cost: views for small/early models, tables
  for moderate marts, **incremental** for large append-mostly tables (with a unique key so re-runs
  upsert, not duplicate — idempotency again). Picking "table" for a billion-row model is the common
  cost mistake; picking "incremental" without a key reintroduces duplicates.

**4. Sources, staging, marts: dbt's project layers map onto the medallion.**
A standard dbt project is layered, mirroring bronze→silver→gold:
  - **Sources** — you *declare* the raw loaded tables (`sources.yml`) so models `ref` them as
    `{{ source('raw','orders') }}`; this also enables **source freshness** checks ("is the raw data
    stale?").
  - **staging** (`models/staging`) — one cleaned, renamed, lightly-typed model per source table.
    Roughly silver's entry.
  - **intermediate** — reusable joins/logic between staging and marts.
  - **marts** (`models/marts`) — the business-facing models: the star schemas and aggregates
    (Chapter 2) your BI reads. Roughly gold.
- *Build consequence:* This structure is the convention interviewers expect: raw stays untouched
  (declared as sources), staging standardizes each source one-to-one, marts hold business logic.
  It keeps logic layered and reusable instead of one giant query, and it's literally the medallion
  expressed in dbt.

**5. Tests turn data quality into code that runs every build.**
dbt has **tests** you declare in YAML and run with `dbt test`:
  - **Generic tests** out of the box: `unique`, `not_null`, `accepted_values` (e.g. status ∈
    {pending, shipped}), and `relationships` (every `order.customer_id` exists in `customers` — a
    referential-integrity check).
  - **Singular/custom tests** — any SQL query that should return *zero* rows (it returns the bad
    rows; non-empty = test fails).

        # schema.yml
        models:
          - name: stg_orders
            columns:
              - name: id
                tests: [unique, not_null]
              - name: status
                tests: [{accepted_values: {values: ['pending','shipped','delivered']}}]

- *Build consequence:* Data quality stops being a manual spot-check and becomes a gate that runs on
  every build/CI — a bad `customer_id` or a duplicate key fails the run *before* it reaches a
  dashboard. This is the dbt half of Chapter 11 (data quality), and a direct answer to "did dbt
  handle data quality?" — yes, via tests.

**6. Jinja and macros: SQL becomes templated, so you stop repeating yourself.**
dbt compiles SQL through **Jinja** (the same templating you've seen in Python web frameworks). That
gives you variables, `if`/`for` loops, and **macros** — reusable SQL functions:

    -- a macro: cents → dollars, defined once, used everywhere
    {% macro to_dollars(col) %}({{ col }} / 100.0){% endmacro %}
    SELECT order_id, {{ to_dollars('amount_cents') }} AS amount_usd FROM {{ ref('stg_orders') }}

`ref()`, `source()`, and tests are themselves Jinja. Macros can range from a one-liner to fairly
complex logic (the bank asks "how complex can macros be?" — quite, but keep them readable).
- *Build consequence:* Repeated SQL patterns (currency conversion, date spines, pivoting a list of
  categories) become a macro written once and reused — DRY SQL. But over-clever macros hurt
  readability; use them to remove genuine repetition, not to show off. Jinja is what makes a dbt
  project scale without copy-paste.

**7. Docs and lineage come for free from the graph.**
Because dbt knows every model, column, and `ref()`, `dbt docs` generates a browsable site with
descriptions *and* a **lineage graph** — a visual DAG of how raw sources flow through staging and
marts to the final tables. This is the *lineage* undercurrent from Chapter 1, automatic.
- *Build consequence:* "Where does this number come from?" and "what breaks if I change this column?"
  are answerable by clicking the lineage graph, not by grepping queries. Documentation stays in sync
  because it's generated from the code, not maintained separately.

**8. dbt in the workflow: orchestrated by Airflow, gated by CI, with snapshots for SCD2.**
dbt computes transformations; it doesn't schedule itself. In production:
  - **Orchestration** — Airflow (Chapter 9) triggers `dbt run`/`dbt test` on a schedule, after
    ingestion lands the raw data. (The bank's "how did you call dbt models through Airflow?")
  - **CI/CD** — on a pull request, a "slim CI" job builds and tests only the changed models (and
    their children) against a sandbox, catching breaking changes *before* merge.
  - **Snapshots** — dbt **snapshots** implement SCD Type 2 (Chapter 2): they capture how a row looked
    over time, with `valid_from`/`valid_to`, automatically.
- *Build consequence:* dbt slots into the platform as the transformation step: ingestion fills
  bronze/raw, Airflow runs dbt to build silver/gold with tests as gates, CI prevents bad models from
  merging, and snapshots handle history. Knowing these seams is what "dbt end-to-end" means in an
  interview.

---

#### Resources (optional — the chapter is self-contained)
- **dbt-duckdb** (`pip install dbt-duckdb`) — a full local dbt project, no warehouse; the hands-on.
- Free: the **dbt "Get started" / Fundamentals** docs and the "Analytics Engineering" guide — read
  for the model/ref/test concepts.

---

#### Hands-on (Python — `pip install dbt-duckdb`)

**A. Stand up a minimal dbt project.** Create this structure (dbt needs a project file and a
profile):

    my_dbt/
      dbt_project.yml         # name: my_dbt; profile: my_dbt
      profiles.yml            # my_dbt: outputs: dev: {type: duckdb, path: dev.duckdb}; target: dev
      models/
        stg_orders.sql        # SELECT * FROM (VALUES (1,1,50),(2,1,20),(3,2,99)) t(id,customer_id,amount)
        customer_orders.sql   # SELECT customer_id, COUNT(*) n, SUM(amount) spent
                              #   FROM {{ ref('stg_orders') }} GROUP BY 1
        schema.yml            # tests: stg_orders.id -> [unique, not_null]

From inside `my_dbt/`, run `dbt run --profiles-dir .` (builds both models in dependency order) then
`dbt test --profiles-dir .` (runs the tests). Confirm both pass and the `customer_orders` table
exists in `dev.duckdb`. Write one sentence on how dbt knew to build `stg_orders` before
`customer_orders`.

**B. Break a test on purpose.** Add a duplicate `id` to `stg_orders` and re-run `dbt test`. Watch the
`unique` test fail and name the offending rows. Write one sentence on why this gate matters before
data reaches a dashboard.

**C. Change a materialization.** Make `customer_orders` `materialized='view'` vs `'table'` (in a
config block) and observe the difference (a view recomputes on read; a table is built once). Write
one sentence on when you'd pick each, and when you'd reach for `incremental` instead.

**D. Reason about layering (no code).** Map a sales pipeline onto sources → staging → marts, and say
which dbt layer corresponds to bronze, silver, and gold, and where you'd put the `relationships` test.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. What does dbt do, and what does "SQL plus software engineering" mean?
2. What is a model, and what does `ref()` do?
3. How does dbt know the order to build models?
4. Name the materializations and when you'd use each.
5. What problem do incremental models solve, and what must they have to stay idempotent?
6. How do sources/staging/marts map to the medallion?
7. Name four generic dbt tests and what each checks.
8. What are Jinja and macros for in dbt?
9. Where do dbt's docs and lineage come from?
10. How does dbt fit with Airflow and CI/CD?

**Apply it (short scenarios — answer in 2–3 sentences):**
11. A warehouse has 200 hand-written SQL scripts with tangled, undocumented dependencies. How does
    moving to dbt fix the order-of-execution and documentation problems?
12. A nightly model rebuilds a 2-billion-row fact table and costs a fortune. What materialization do
    you switch to, and what do you need for it to be correct?
13. A bad upstream change put duplicate keys and an invalid status into a mart last week and nobody
    noticed until a report was wrong. What dbt features would have caught it, and when?
14. The same currency-conversion expression is copy-pasted across 15 models. What dbt feature removes
    the duplication?
15. A view's SQL hard-codes `dev.raw.orders`, so deploying to prod breaks. What's the dbt-correct way
    to reference it so dev and prod both work?

**Stretch / discussion (optional):**
16. How do dbt snapshots relate to SCD Type 2 from Chapter 2?
17. What is "slim CI," and why build only changed models and their children rather than everything?

**Answer key (peek only after attempting):**
1. Transforms data in the warehouse from SQL SELECTs while managing DDL, build order, tests, docs,
and version control. · 2. A `.sql` file with one SELECT defining a table/view; `ref()` references
another model and creates the dependency. · 3. From the `ref()`s it builds a DAG and runs models in
dependency order. · 4. view (recompute on read, cheap), table (rebuild fully, fast reads),
incremental (process only new rows), ephemeral (inlined like a CTE). · 5. They avoid rebuilding huge
tables by processing only new/changed rows; they need a unique key so re-runs upsert instead of
duplicating. · 6. Sources = raw (bronze), staging = one cleaned model per source (silver entry),
marts = business models/star schemas (gold). · 7. unique, not_null, accepted_values, relationships
(referential integrity). · 8. Templating SQL — variables/loops and reusable macros to remove
repetition (DRY). · 9. Auto-generated from the models, columns, and `ref()` graph (`dbt docs`),
including a lineage DAG. · 10. Airflow schedules `dbt run`/`dbt test` after ingestion; CI builds/tests
changed models on PRs before merge. · 11. `ref()` derives the build order automatically and
`dbt docs` generates lineage/descriptions from the code. · 12. Incremental materialization with a
unique key (and an incremental filter), so it processes new rows and upserts rather than rebuilding.
· 13. `unique`/`not_null` and `accepted_values` tests run by `dbt test` in CI/nightly would have
failed the build before the report. · 14. A macro, defined once and called in all 15 models. · 15.
Use `{{ source('raw','orders') }}` (or `ref`), so dbt resolves the right schema per environment. ·
16. Snapshots implement SCD2 — they record row versions over time with valid_from/valid_to
automatically. · 17. A CI job that builds/tests only models changed in the PR plus their downstream
children, so it's fast and still catches breakage without rebuilding the whole project.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Explain dbt and describe a project you did with it (end to end).
2. Explain incremental models in dbt and when to use them.
3. Describe a typical dbt folder structure (staging/intermediate/marts) and what goes in each, plus
   where sources and tests are defined.
4. Did dbt handle data quality checks? What transformations did you build, and how?
5. How complex can dbt macros be?
6. How did you call dbt models through Airflow?
7. Describe end-to-end CI for dbt (env, seed data, tests, slim CI) to catch breaking changes pre-merge.
8. How are materialized views analogous to CTEs (and to ephemeral models)?
9. How do you handle a model that must not rebuild fully every run?
10. Where do you define sources, and what does source freshness give you?
11. How do you reference another model so dev/prod schemas resolve correctly?
12. How would you test referential integrity between two models?
13. Compare doing transformations in dbt vs in PySpark — when would you pick each?
14. How do dbt snapshots implement slowly changing dimensions?
15. A complex transformation needs error handling on schema validation — how do you approach it in dbt?

_(dbt ~78, transformation ~55, incremental/materialization ~14, tests/snapshots/seeds ~8, Jinja/
macros ~7 questions in the bank; 15 shown — more in [data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** A runnable dbt-duckdb project with at least two models connected by `ref()` and one
passing test, plus a screenshot/printout of `dbt run` and `dbt test` succeeding; then deliberately
break the test and capture the failure. Add a one-paragraph note mapping your models to the medallion
layers. Attach your answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~4–5 hours.

---
