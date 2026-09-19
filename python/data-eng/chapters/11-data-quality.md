## Chapter 11 — Data Quality & Reliability: Shipping Pipelines with Confidence

**Goal:** Learn to make pipelines *trustworthy* — the dimensions of data quality, the concrete checks
you encode (null/unique/valid/referential/volume), deduplication, where checks live and what to do
on failure, SLAs/SLOs and data contracts, and the observability + DataOps loop that keeps a running
platform healthy.

**What we assume you know:** SQL (Chapter 2), dbt tests (Chapter 8), the medallion boundaries
(Chapter 5), idempotency and dead-lettering (Chapters 1/6). This chapter ties them into a quality
discipline.

**Why this matters:** A pipeline that runs but produces *wrong* data is worse than one that
fails — because people trust the numbers and act on them. Data quality is what separates "the
dashboard is up" from "the dashboard is right." It's a heavily-asked area because every real platform
has been burned by a silent data bug.

> **Setup assumed:** Python with DuckDB for the hands-on (the checks are SQL you can run anywhere).

**Suggested split across two working sessions:**
- **Session 1** — concepts 1–4: why quality, the dimensions, checks-as-code, deduplication.
- **Session 2** — concepts 5–8: where checks live + failure handling, tooling, SLAs/SLOs & contracts,
  observability/DataOps + hands-on.

---

#### Core concepts

**1. Bad data is worse than no data, because it's trusted.**
A crashed pipeline pages someone and gets fixed. A pipeline that quietly loads *wrong* data flows
straight to a dashboard, a board deck, or an ML model — and nobody notices until a decision was made
on it. "Garbage in, garbage out," but the garbage wears a suit. Data quality is the discipline of
catching bad data *before* it's trusted.
- *Build consequence:* You build *verification into the pipeline*, not as an afterthought — the same
  way Chapter 1 said design for the undercurrents from day one. Every pipeline needs an answer to
  "how would I know if this data went wrong?" before it ships, not after the incident.

**2. The dimensions of data quality — name them so you know what to check.**
"Quality" is vague until you break it into measurable dimensions:
  - **Completeness** — is anything missing? (nulls in required fields, missing days, dropped rows)
  - **Uniqueness** — duplicates? (a key appearing twice)
  - **Validity** — does each value obey its rules? (status ∈ allowed set; amount > 0; valid email)
  - **Accuracy** — does it match reality? (hardest; often checked by reconciling totals to a source)
  - **Consistency** — do related sources agree? (order count in two systems matches)
  - **Timeliness / freshness** — is it up to date? (today's data arrived by 6 a.m.)
- *Build consequence:* For each table, decide which dimensions matter and write a concrete check per
  one. "Check data quality" becomes a checklist — not a vibe — which is exactly what the bank's
  "what do you check?" question wants: completeness, uniqueness, validity, freshness, and reconciled
  accuracy.

**3. Checks as code: encode each dimension as a query that fails loudly.**
Each dimension maps to a concrete, automatable check (the dbt tests of Chapter 8 are these, packaged):
  - **not-null** — `COUNT(*) WHERE key IS NULL` must be 0 (completeness).
  - **unique** — `GROUP BY key HAVING COUNT(*) > 1` returns no rows (uniqueness).
  - **accepted values / range** — `WHERE status NOT IN (...)` or `WHERE amount <= 0` returns no rows
    (validity).
  - **referential integrity** — every `fact.customer_id` exists in `dim_customer` (no orphans).
  - **volume / row-count anomaly** — today's row count is within an expected band of recent days (a
    sudden 10× or 0.1× signals a broken upstream).
- *Build consequence:* Quality checks are just queries that should return "zero bad rows," run
  automatically on every load. Encode them as code (dbt tests, SQL assertions) so they run every time
  and fail the build — not as a human eyeballing a sample once.

**4. Deduplication: the most common quality fix — find them, then keep the right one.**
Duplicates are the bread-and-butter quality problem (and a frequent bank question). Two steps:
  - **Find** them: `SELECT key, COUNT(*) FROM t GROUP BY key HAVING COUNT(*) > 1`.
  - **Remove** them: keep one row per key — usually the *latest* — with the Chapter 2 window pattern
    `ROW_NUMBER() OVER (PARTITION BY key ORDER BY updated_at DESC) ... WHERE rn = 1`. (Careful with
    "rows where id+name match but address differs" — decide which is canonical before dropping.)
- *Build consequence:* Dedup at the silver boundary (Chapter 5/6), choosing the survivor
  deliberately (latest by timestamp, or a business rule) — and prevent re-introduction by making the
  load idempotent (MERGE on the key). Blindly `DISTINCT`-ing hides the real question of *which* row is
  correct.

**5. Where checks live and what to do when one fails: gate, quarantine, or warn.**
Place checks at the boundary where bad data would otherwise spread — **entering silver** (Chapter 5's
schema-enforced boundary). When a check fails, you have options:
  - **Gate (fail the pipeline)** — stop and alert, so bad data never reaches gold. Right for critical
    checks (a null primary key).
  - **Quarantine / dead-letter** — route bad rows to a side table and let the good ones through, so
    one bad batch doesn't block everything (Chapter 6's dead-letter, applied to quality).
  - **Warn** — log/alert but proceed, for non-critical anomalies you want visibility on.
- *Build consequence:* Decide per check whether a failure should *stop* the pipeline (data integrity
  at stake) or *quarantine* the bad rows (keep flowing, fix the few). Putting checks at the silver
  boundary means gold — what users see — is only ever built from validated data.

**6. Tooling: dbt tests, Great Expectations, Delta constraints — build vs buy.**
You don't hand-roll everything:
  - **dbt tests** (Chapter 8) — the easiest if you're already transforming in dbt; `unique`,
    `not_null`, `accepted_values`, `relationships`, plus custom.
  - **Great Expectations / Soda** — dedicated data-quality frameworks: declare "expectations" (a
    column's values are in a range, X% non-null), get validation runs and reports/docs.
  - **Delta/warehouse constraints** — `NOT NULL`/`CHECK` constraints enforced by the table itself.
- *Build consequence:* If you transform in dbt, start with dbt tests (least new infrastructure); reach
  for Great Expectations/Soda when you need richer profiling, validation docs, or checks outside the
  transformation tool. The point is the checks run automatically and are versioned — not which tool.

**7. SLAs, SLOs, and data contracts: agree on targets and on the data's shape.**
The bank's reliability question. Borrowed from software ops:
  - **SLO (objective)** — a target you hold yourself to: "the sales table is fresh by 6 a.m. 99% of
    days," "row-count within ±5% of the 7-day average."
  - **SLA (agreement)** — a commitment to a consumer, often with consequences, built on SLOs.
  - You **measure** these (freshness lag, % checks passing, completeness rate) and **report** them on
    a dashboard.
  - **Data contract** — an explicit agreement with the *producer* on schema and semantics (fields,
    types, meaning, guarantees), so schema drift (Chapter 6) becomes a contract violation you can
    detect and push back on, not a silent break.
- *Build consequence:* Define a few concrete SLOs per important dataset (freshness, completeness),
  measure and publish them, and where possible formalize a data contract with upstream so changes are
  negotiated, not sprung on you. "Define and report SLAs/SLOs" = pick measurable targets (freshness,
  quality pass-rate), instrument them, show them.

**8. Observability and the DataOps loop: detect → triage → fix → prevent.**
Quality isn't one-and-done; a running platform needs **observability** — continuous monitoring of
the signals that reveal trouble: **freshness** (is data arriving on time?), **volume** (row counts in
range?), **schema** (did it change?), **distribution** (did a column's values shift?), and
**lineage** (what's downstream of a broken table?). When something fires, the **DataOps loop** runs:
detect (an alert), triage (what broke, what's affected — lineage), fix (the right artifact), and
prevent (add a check so it can't recur).
- *Build consequence:* Instrument the four golden signals (freshness, volume, schema, distribution)
  with alerts, use lineage to scope the blast radius of an incident, and close the loop by adding a
  check after every incident so the same bug can't return silently. This is how a platform stays
  trustworthy over time, not just on launch day. (The Advanced track's monitoring/drift chapter goes
  deeper.)

---

#### Resources (optional — the chapter is self-contained)
- **DuckDB** for the hands-on quality checks.
- **dbt tests** (Chapter 8) you've already run — the simplest quality gate.
- Free reads for shape: the Great Expectations "core concepts" intro and any "data observability /
  five pillars" overview (freshness, volume, schema, distribution, lineage).

---

#### Hands-on (Python — DuckDB)

**A. Run the core quality checks (each should return "0 bad rows").**

    import duckdb
    con = duckdb.connect()
    con.execute("""CREATE TABLE orders AS SELECT * FROM (VALUES
      (1,'shipped',50),(2,'shipped',20),(2,'pending',20),(3,'unknown',-5),(4,'pending',NULL)
    ) t(id,status,amount)""")
    checks = {
      "null_amount":   "SELECT COUNT(*) FROM orders WHERE amount IS NULL",
      "dup_id":        "SELECT COUNT(*) FROM (SELECT id FROM orders GROUP BY id HAVING COUNT(*)>1)",
      "bad_status":    "SELECT COUNT(*) FROM orders WHERE status NOT IN ('pending','shipped','delivered')",
      "nonpos_amount": "SELECT COUNT(*) FROM orders WHERE amount <= 0",
    }
    for name, q in checks.items():
        bad = con.execute(q).fetchone()[0]
        print(f"{name}: {'PASS' if bad==0 else f'FAIL ({bad} bad rows)'}")

1. Run it; you'll see several FAIL. For each failing check, name the *dimension* of quality it maps to
   (completeness / uniqueness / validity).

**B. Deduplicate to the latest per key.** Add an `updated_at` column; for the duplicate `id=2`, keep
the latest with `ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) ... WHERE rn=1`. Confirm
one row per id. Write one sentence on how you'd prevent the dup from coming back (idempotent load).

**C. Freshness / volume check (no code or simple).** Given a table with a `loaded_at` and daily row
counts, write the check for "data is fresh (max loaded_at within 24h)" and "today's count is within
±50% of the 7-day average." Write one sentence on what a sudden 10× row count usually means.

**D. Reason about response & SLOs (no code).** For each check in A, decide: **gate** (stop pipeline),
**quarantine** (side-table the bad rows, let good through), or **warn**. Then define two SLOs for this
orders table (one freshness, one quality) and say how you'd measure each.

---

#### Questions

**Check your understanding (answer in a sentence each):**
1. Why is bad data worse than a failed pipeline?
2. Name five dimensions of data quality with a one-line check for each.
3. How do you express a "unique" and a "not-null" check as SQL that returns zero bad rows?
4. What is a referential-integrity check?
5. What is a volume/row-count anomaly check good for?
6. The two steps to deduplicate, and how you pick the survivor?
7. Where should quality checks live, and what are the three failure responses?
8. Name two data-quality tools and when you'd use each.
9. SLA vs SLO vs data contract — one line each.
10. Name the four "golden signals" of data observability and the DataOps loop steps.

**Apply it (short scenarios — answer in 2–3 sentences):**
11. A revenue dashboard was wrong for a week before anyone noticed. What checks and signals would
    have caught it, and where would they run?
12. A daily load occasionally double-counts revenue. Name the find-and-fix and how you prevent
    recurrence.
13. One bad batch (10 malformed rows out of 1M) keeps failing the whole nightly load. How do you keep
    the pipeline flowing without ignoring the problem?
14. A client wants "clear SLAs/SLOs" for a pipeline. Walk through what you'd define, measure, and
    report.
15. Upstream silently changed a field's type and broke gold. What two mechanisms (one preventive, one
    detective) should have been in place?

**Stretch / discussion (optional):**
16. Accuracy is the hardest dimension to check. How might you approximate an accuracy check without a
    perfect source of truth?
17. After an incident, what does "close the loop" mean, and why is it the most important step?

**Answer key (peek only after attempting):**
1. A failure is visible and gets fixed; wrong data is trusted and acted on before anyone notices. ·
2. Completeness (no nulls in required fields), uniqueness (no duplicate keys), validity (values obey
rules), accuracy (matches reality/reconciles), timeliness (fresh by deadline). · 3. unique:
`GROUP BY key HAVING COUNT(*)>1` returns no rows; not-null: `COUNT(*) WHERE col IS NULL` = 0. · 4.
Every foreign key value (e.g. `fact.customer_id`) exists in the referenced dimension — no orphans. ·
5. Catching a broken upstream when today's volume is far outside the normal band (a 10× spike or near
zero). · 6. Find with `GROUP BY ... HAVING COUNT(*)>1`; remove by keeping one per key (latest via
ROW_NUMBER), choosing the survivor by timestamp/business rule. · 7. At the silver boundary; respond by
gating (stop), quarantining (side-table bad rows, pass good), or warning. · 8. dbt tests (if
transforming in dbt) and Great Expectations/Soda (richer profiling/validation docs/standalone). · 9.
SLO = a target you hold yourself to; SLA = a commitment to a consumer; data contract = an agreement
with the producer on schema/semantics. · 10. Freshness, volume, schema, distribution (+ lineage); loop
= detect → triage → fix → prevent. · 11. Freshness, volume, validity, and reconciliation checks at the
silver boundary plus alerting would have flagged it within a day. · 12. Find dups via GROUP BY HAVING;
keep latest per key; prevent with an idempotent MERGE on the key. · 13. Quarantine/dead-letter the 10
bad rows to a side table and let the good rows through, then fix/alert on the quarantine. · 14. Define
freshness and quality-pass SLOs, formalize them into an SLA, instrument freshness lag and % checks
passing, and report on a dashboard. · 15. Preventive: a data contract with the producer; detective: a
schema/freshness check that fails the build on drift (Chapter 6). · 16. Reconcile aggregates against a
trusted source (totals match the source system), use range/distribution checks, and spot-audit
samples. · 17. Add a check/monitor so that exact failure can't recur silently — it converts a one-time
incident into permanent coverage, which is what steadily raises platform trust.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. How do you implement data quality checks in a pipeline, and what do you check?
2. Describe a case where data quality checks uncovered a critical data issue.
3. Did you automate data quality? How, and did you create quality metrics?
4. Describe intermediate-level data quality checks you've built.
5. A client needs clear SLAs/SLOs — how do you define, measure, and report reliability and quality?
6. Find duplicate email addresses in a users table (id, email).
7. Give syntax to deduplicate in both Python and SQL for a customer table.
8. Get the latest record for each id from a table of (id, status, timestamp).
9. How do you handle duplicate records or junk characters in data validation?
10. Explain data cleansing and data quality (e.g. in GCP).
11. How can big data achieve high observability?
12. Describe a scenario where a data quality rule was slow and how you optimized it.
13. How does testing work in dbt, and what tests have you written?
14. Explain a strategy that improved data governance and its impact on data quality.
15. Reconstruct the latest snapshot from a landing zone with updates and deletes over time.

_(data quality ~40, monitoring/observability ~33, validation/completeness/accuracy ~26 questions in
the bank; 15 shown — more in [data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** A DuckDB script that runs the four core checks (printing PASS/FAIL with bad-row
counts) and a dedup-to-latest, plus a short note defining two SLOs (freshness + quality) for a table
and, per check, whether failure should gate or quarantine. Attach your answers to questions 1–15.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~4–5 hours.

---
