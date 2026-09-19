## Chapter 9 — Orchestration with Airflow: Scheduling the Whole Graph

**Goal:** Understand orchestration — why a pipeline is a graph of dependent tasks that something must
schedule, sequence, retry, and backfill — and how Airflow does it: DAGs, operators, scheduling and
the data interval, retries/SLAs, backfills, and parallelism.

**What we assume you know:** Python (Airflow DAGs are Python), and Chapters 1–8 — especially
idempotency (Ch1), the DAG idea (Ch4/8), and that ingestion/dbt/Spark are the *steps* something has
to run in order.

**Why this matters:** Every real pipeline is many steps with dependencies ("load raw, then build
silver, then gold, then refresh the dashboard") that must run on a schedule, recover from failures,
and be re-runnable for past dates. Orchestration is the undercurrent (Chapter 1) that ties all the
other chapters into a running system. Airflow is the most common tool for it.

> **Setup assumed:** conceptual + a pure-Python simulation for the hands-on (no Airflow install).
> Real Airflow runs via Docker (`astro`/`docker compose`); the DAG *code* you write is plain Python.

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: why orchestration, the DAG, tasks/operators/dependencies, scheduling.
- **Session 2** — concepts 5–8: retries/SLAs, backfills & idempotency, parallelism, Airflow's place
  + hands-on.

---

#### Core concepts

**1. Orchestration: a pipeline is a graph of dependent tasks, and something must run them correctly.**
You could chain steps with cron + shell scripts — `cron: run ingest.py, then transform.py`. It falls
apart fast: cron has no notion of *dependencies* (transform must wait for ingest to *succeed*), no
*retries*, no *backfill*, and no *visibility* (which step failed, when, why?). **Orchestration** is a
system that runs a graph of tasks in dependency order, on a schedule, with retries, monitoring, and
the ability to re-run history.
- *Build consequence:* The moment a pipeline has more than one step with an ordering dependency, you
  want an orchestrator, not cron + glue. It's what turns "a pile of scripts" into "a pipeline you can
  operate" — the difference is dependencies, retries, and visibility.

**2. The DAG: your pipeline as a directed acyclic graph of tasks, written in Python.**
Airflow models a pipeline as a **DAG** — Directed (dependencies point one way), Acyclic (no loops, so
it always terminates), Graph (of tasks). You define it in **Python**, which is the part that clicks
for you: tasks are like functions, dependencies are arrows.

    ingest >> transform_silver >> build_gold >> refresh_dashboard
    # build_gold runs only after transform_silver succeeds; the arrows ARE the dependencies

"Acyclic" matters: a cycle (A waits for B waits for A) could never start, so Airflow forbids it.
- *Build consequence:* You express *what depends on what*, not *when to run each thing* — the
  scheduler derives the order from the graph (just like dbt's `ref()`, Chapter 8). Define
  dependencies correctly and ordering, parallelism, and "don't start gold until silver succeeded"
  all come for free.

**3. Tasks, operators, and dependencies: what each step does, and how they connect.**
  - A **task** is one node — one unit of work.
  - An **operator** defines *what kind* of work: `PythonOperator` (run a function), `BashOperator`
    (a shell command), SQL/Spark/Databricks operators, and many provider-specific ones (run dbt,
    trigger a Databricks job).
  - **Dependencies** wire tasks: `a >> b` ("a then b"), and a task can have many parents/children
    (fan-in / fan-out).
- *Build consequence:* You pick the operator that matches each step (a `DatabricksRunNowOperator` to
  run a notebook, a `BashOperator` to run `dbt`), and wire dependencies so each step waits for what
  it truly needs. Airflow *orchestrates* these — the heavy lifting still happens in Spark/dbt/the
  warehouse; Airflow just runs them in order (concept 8).

**4. Scheduling and the data interval: each run processes a time window, not "now."**
A DAG has a **schedule** (a cron expression or preset like `@daily`). The subtle, important idea:
each run is tied to a **data interval** (historically `execution_date`) — the *time window of data*
that run is responsible for. The `@daily` run "for 2026-06-17" processes that day's data and
typically fires *after* the day ends. **Catchup** controls whether Airflow runs all missed intervals
since the start date (e.g. to fill history) or just the latest.
- *Build consequence:* Parameterize every task by its data interval (`WHERE day = {{ data_interval
  _start }}`), not by "today" — so a run for any date processes *that date's* data. This is what makes
  backfills (concept 6) correct, and it's why hard-coding `today()` inside a task is a classic bug
  that breaks re-runs.

**5. Retries, failures, and SLAs: assume tasks fail, and make recovery automatic.**
Tasks fail — a source is briefly down, a cluster hiccups. Airflow gives each task:
  - **retries** with a delay (and backoff) — a transient failure auto-retries instead of paging you.
  - **failure handling** — `on_failure_callback` to alert (Slack/email), and downstream tasks are
    skipped/blocked so bad data doesn't propagate.
  - **SLAs** — declare "this task should finish within N hours"; Airflow flags a miss, which feeds the
    SLA/SLO reporting the bank asks about.
- *Build consequence:* Set sensible retries (with backoff) on every task and alerting on failure, so
  the pipeline self-heals transient issues and tells you about real ones — combined with idempotency
  (concept 6), a retry is always safe. This is Chapter 6's reliability, now at the orchestration
  level.

**6. Backfills and idempotency: re-running history must be safe — which is why tasks must be idempotent.**
Because each run owns a data interval, you can **backfill** — run the DAG for a *range* of past dates
(you added a new gold table and need 2 years of history; or a bug corrupted last week). Airflow will
happily run "2024-01-01" through "today." But backfill *re-executes* tasks — so every task must be
**idempotent** (Chapter 1): running "2026-06-17" twice must leave the same state (MERGE / partition-
overwrite that date, never blind append). Re-running a DAG run ("clear and rerun") relies on the same
property.
- *Build consequence:* Idempotency isn't optional once you have an orchestrator — backfills and
  retries *will* re-run tasks, and only idempotent + interval-parameterized tasks survive that
  without duplicating or corrupting data. "How do you rerun a DAG?" really tests "are your tasks
  safe to re-run?"

**7. Parallelism: independent tasks run at the same time — the graph decides what can.**
Airflow runs tasks with no dependency between them **in parallel**, across workers (the executor —
Celery/Kubernetes in production). Fan-out: after `ingest`, build `dim_customer`, `dim_product`, and
`dim_date` simultaneously, then fan-in to `fact_sales` once all three finish. **Pools** and
concurrency limits cap how much runs at once (so you don't overwhelm a database or cluster).
- *Build consequence:* Structure the DAG so independent work is actually independent (parallel
  branches) rather than a needless straight line — it cuts wall-clock time. But bound parallelism
  with pools when tasks hit a shared limited resource (a source DB's connection limit), or you trade
  a slow pipeline for an overloaded source.

**8. Airflow orchestrates; it doesn't process — and it has alternatives.**
A crucial distinction: Airflow **triggers and sequences** work; it does not crunch the data itself.
The right pattern is "thin" tasks that kick off heavy work elsewhere — trigger a Databricks/Spark job,
run `dbt`, run a warehouse query — and let *that* engine do the processing. **Sensors** are special
tasks that *wait* for a condition (a file landed, an upstream table is ready) before downstream runs.
Alternatives exist and get compared in interviews: **Dagster** and **Prefect** (more
asset/data-aware, modern DX), **Azure Data Factory** (managed, our cloud's native orchestrator),
**Argo/Kubeflow** (Kubernetes-native).
- *Build consequence:* Don't load 100 GB *into Airflow* and transform it there — that melts the
  scheduler. Make tasks thin (trigger Spark/dbt, wait on a sensor, move metadata) and let the
  processing engines do the work. Choose the orchestrator for the environment: Airflow for
  flexible/code-first, ADF on an all-Azure stack, Dagster when you want data-asset awareness.

---

#### Resources (optional — the chapter is self-contained)
- Pure-Python simulation (below) — learn DAG ordering, cycles, and idempotent backfill without
  installing Airflow.
- Stretch: **Airflow via Docker** (the official `docker compose` or Astronomer's `astro dev`) to see
  the UI, DAG graph, and a backfill for real.
- Free reads for shape: the Airflow "Core Concepts" docs (DAGs, operators, scheduling) and "Backfill
  and Catchup."

---

#### Hands-on (pure Python — models the scheduler; no Airflow needed)

**A. Build a DAG and compute its run order (topological sort).**

    deps = {                              # task -> its upstream dependencies
        "ingest": [],
        "dim_customer": ["ingest"], "dim_product": ["ingest"], "dim_date": ["ingest"],
        "fact_sales": ["dim_customer", "dim_product", "dim_date"],
        "refresh_dashboard": ["fact_sales"],
    }
    def run_order(deps):
        done, order = set(), []
        while len(done) < len(deps):
            ready = [t for t, ups in deps.items() if t not in done and all(u in done for u in ups)]
            if not ready: raise ValueError("cycle or missing dep!")
            order.extend(sorted(ready)); done.update(ready)     # 'ready' tasks could run in parallel
        return order
    print(run_order(deps))

1. Run it. Note that the three dims become "ready" together — those run in **parallel** in Airflow.
   Write one sentence on how the graph (not a manual schedule) produced this order.
2. Add a cycle (`ingest` depends on `refresh_dashboard`) and confirm it raises — that's why DAGs must
   be acyclic.

**B. Idempotent task parameterized by execution date.** Write a `load(day, store)` that
partition-overwrites `store[day]` (a dict) rather than appending. Run it twice for `'2026-06-17'` and
confirm the state is identical — then "backfill" a range of three days and confirm re-running the
whole range changes nothing. Write one sentence linking this to why backfill needs idempotency.

**C. Read a real Airflow DAG (no run).** Study and annotate:

    with DAG("sales", schedule="@daily", catchup=False, default_args={"retries": 2}) as dag:
        ingest   = BashOperator(task_id="ingest", bash_command="python ingest.py {{ ds }}")
        dbt_run  = BashOperator(task_id="dbt", bash_command="dbt run --profiles-dir .")
        ingest >> dbt_run
    # mark: where are the schedule, retries, the data-interval ({{ ds }}), and the dependency?

**D. Reason about design (no code).** (a) Why make the `dbt` task *trigger* dbt rather than do the
transformation in Python inside Airflow? (b) Where would a **sensor** go in a pipeline that must wait
for a file to land? (c) When would you pick ADF over Airflow?

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. Why isn't cron + shell scripts enough for a multi-step pipeline?
2. What does DAG stand for, and why must it be acyclic?
3. What's the difference between a task and an operator?
4. What is the data interval / execution date, and why parameterize tasks by it?
5. What does catchup control?
6. How do retries and SLAs help, and what makes a retry safe?
7. What is a backfill, and why does it require idempotent tasks?
8. How does Airflow decide what can run in parallel, and what bounds it?
9. Why should Airflow tasks be "thin," and what should do the heavy processing?
10. What is a sensor, and name one Airflow alternative and when you'd use it.

**Apply it (short scenarios — answer in 2–3 sentences):**
11. A task hard-codes `today()` to filter data; backfilling 2023 produces wrong results. What's the
    bug and the fix?
12. You add a new gold table and need 18 months of history. What Airflow feature do you use, and what
    property must the tasks have for it to be correct?
13. Three dimension builds are independent but your DAG runs them in a straight line, making the
    pipeline slow. What do you change, and what might you then need to add to protect the source DB?
14. A flaky source causes nightly failures that page you at 3 a.m.; the data is fine on a re-run. How
    do you stop the pages without ignoring real failures?
15. Someone proposes reading a 200 GB table into a PythonOperator and transforming it in Airflow. Why
    is that wrong, and what's the right pattern?

**Stretch / discussion (optional):**
16. Compare Airflow and Dagster (or ADF) — when would you choose each?
17. How do dbt and Airflow fit together in one project, and what does each own?

**Answer key (peek only after attempting):**
1. Cron has no dependency awareness, retries, backfill, or visibility into which step failed. · 2.
Directed Acyclic Graph; acyclic so it always terminates (a cycle could never start). · 3. A task is
one node/unit of work; an operator defines what kind of work it does (Python, Bash, SQL, etc.). · 4.
The time window of data a run owns; parameterizing by it makes each run process the right date and
makes backfills correct. · 5. Whether Airflow runs all missed intervals since the start date or only
the latest. · 6. Retries auto-recover transient failures and SLAs flag slow tasks; a retry is safe
when the task is idempotent. · 7. Running the DAG for a range of past dates; it re-executes tasks, so
they must be idempotent to avoid duplicates/corruption. · 8. Tasks with no dependency between them
run in parallel across workers; pools/concurrency limits cap it. · 9. Airflow orchestrates, it
doesn't process; trigger Spark/dbt/warehouse jobs and let those engines crunch the data. · 10. A task
that waits for a condition (file/table ready); e.g. Dagster (data-asset aware) or ADF (Azure-native)
— choose per environment. · 11. The task isn't parameterized by the data interval; use
`{{ data_interval_start }}`/`{{ ds }}` so each run filters its own date. · 12. Backfill over the date
range; tasks must be idempotent (and interval-parameterized) so re-running doesn't duplicate. · 13.
Make the three dims parallel branches that fan-in to the fact; add a pool/concurrency limit so they
don't exhaust the source DB's connections. · 14. Add retries with backoff (auto-recover transient
issues) and alert only on final failure; idempotency keeps the retry safe. · 15. It overwhelms the
scheduler; Airflow should trigger a Spark/warehouse job that does the processing, keeping the task
thin. · 16. Airflow = flexible code-first orchestration; Dagster = data-asset/lineage aware with
better local DX; ADF = managed and native on Azure — pick per stack and team. · 17. dbt defines/builds
and tests the transformations; Airflow schedules and sequences `dbt run`/`dbt test` after ingestion
and triggers other jobs — Airflow owns *when/order*, dbt owns *the transformations*.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Explain what a data pipeline is and why orchestration is important for managing pipelines.
2. How did you use Airflow in your project — challenges and solutions?
3. How does a DAG look in Airflow?
4. How does Airflow handle parallelism in DAGs?
5. How do you implement event triggers in Airflow and manage task dependencies and parallelism?
6. How do you rerun a DAG, and how do you add kill functionality to a DAG?
7. How do you handle retries and failures in ETL pipelines?
8. How did you call dbt models through Airflow?
9. How did you use dbt, Data Factory, BigQuery, and Airflow together in one project?
10. How do you monitor a workflow in Airflow?
11. Compare Dagster and Airflow — when to choose which?
12. Compare Kubeflow, Airflow, and Argo Workflows — when to use what?
13. How can you trigger a pipeline in Data Factory when source data is updated?
14. Define SLAs/SLOs for a pipeline and explain how you'd measure and report them.
15. Describe a job-orchestration project you worked on, end to end.

_(Airflow ~44, orchestration ~16, DAG ~13, schedule/cron/backfill ~11, retries ~6, sensor/operator
~5 questions in the bank; 15 shown — more in [data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** The Python DAG simulation (A + B): a `run_order` that produces a correct
topological order and detects a cycle, and an idempotent date-parameterized task proving a 3-day
backfill is safe to re-run. Plus an annotated real Airflow DAG identifying the schedule, retries,
data interval, and dependencies. Attach your answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~4–5 hours.

---
