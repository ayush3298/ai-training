## Advanced Track — DataOps, CI/CD & Infrastructure-as-Code

**Goal:** Bring software-engineering discipline to the *operation* of data pipelines — testing,
environments (dev→staging→prod), CI/CD, infrastructure-as-code, and the monitor→incident→improve
loop — so changes ship safely and the platform stays healthy after launch day.

**What we assume you know:** Chapters 1–11, especially dbt tests/CI (Chapter 8), Airflow (Chapter 9),
quality/observability (Chapter 11), and the software-engineering undercurrent (Chapter 1).

**Why this matters:** A pipeline is software, and most teams run it without software discipline — no
tests, manual deploys, prod-only changes, no monitoring. DataOps terms appear ~70 times in the bank,
and "how do you deploy / handle environments / catch breaking changes" separates engineers who *ship*
from those who *hack*.

> **Setup assumed:** conceptual; builds on the dbt project (Chapter 8) and the capstone repo. Bring
> the capstone to apply CI and environments to it.

---

#### Core concepts

**1. DataOps: apply DevOps to data — version, test, automate, monitor.**
**DataOps** is the practice of running data pipelines like software: everything in **version control**
(SQL, dbt models, DAGs, configs, infra), automated **testing**, automated **deployment**, and
continuous **monitoring**. The opposite — editing queries in a prod console, no tests, no history — is
how silent data bugs and 3 a.m. outages happen.
- *Build consequence:* If it's not in git, it doesn't exist. Treat pipeline code, transformations,
  and infrastructure as versioned artifacts that flow through an automated, tested release process —
  not living configurations someone tweaks in prod.

**2. Environments: dev → staging → prod, with isolation.**
You don't develop against production data and tables. Standard separation: **dev** (your sandbox),
**staging/test** (prod-like, for integration tests), **prod** (the real thing). Each has its own
schemas/storage/compute, and code is *promoted* between them. This is exactly Chapter 5's
"no dev-specific schema in prod" problem — solved by parameterizing the target per environment (dbt
targets, Chapter 8's `ref()`/`source()` resolving schemas per env).
- *Build consequence:* Parameterize every table reference by environment so the same code runs in dev
  and prod against different schemas; never hard-code a schema/catalog. Promote code through
  environments rather than editing prod directly — so a mistake is caught in staging, not by users.

**3. Testing pipelines: unit, data, and integration tests.**
Pipelines need layered tests:
  - **Unit tests** — test transformation logic on small fixed inputs (a Python/Spark function; dbt
    unit tests).
  - **Data tests** — the quality checks (Chapter 11): unique, not-null, accepted values, referential
    — run on the actual data each build.
  - **Integration tests** — run the pipeline end-to-end on seed data in staging and assert the output.
- *Build consequence:* Combine logic tests (does the code compute correctly on known inputs?) with
  data tests (is the real data valid?) and an end-to-end run on seed data. The first catches code
  bugs, the second catches data bugs — you need both, and CI runs them automatically.

**4. CI/CD: build, test, and deploy automatically — with "slim" data CI.**
**CI** (continuous integration) runs on every pull request: build the changed models, run tests, fail
the PR if anything breaks — catching problems *before* merge. **CD** (continuous deployment) promotes
the merged code to prod automatically. For dbt, **slim CI** builds/tests *only the changed models and
their downstream children* against a sandbox, so CI is fast and still catches breakage (Chapter 8).
- *Build consequence:* Set up CI so a PR that breaks a model or fails a test can't merge, and CD so
  deploys are automated and repeatable (not a manual checklist). Slim CI keeps it fast on large
  projects by testing only the blast radius of the change.

**5. Infrastructure-as-Code: define clusters, warehouses, and jobs in versioned config.**
Don't click resources into existence in a console — define them in code (**Terraform**, or
Databricks Asset Bundles / cloud-native templates). The infra (clusters, warehouses, storage,
permissions, scheduled jobs) becomes versioned, reviewable, and reproducible across environments.
- *Build consequence:* IaC makes environments identical and rebuildable — spin up staging matching
  prod from the same code, review infra changes in PRs, and recover from "someone changed a setting"
  by re-applying. Manually-configured infra is unreproducible and drifts; codified infra doesn't.

**6. The monitor → incident → improve loop: operating after launch.**
Shipping isn't done; the platform runs continuously. The operational loop (Chapters 9, 11): **monitor**
the golden signals (freshness, volume, schema, distribution) + pipeline health (failures, durations,
SLA misses); on an **incident**, alert, use lineage to scope impact, fix the right artifact, and
communicate; then **improve** — add a test/monitor so it can't recur, and feed recurring pain into the
backlog. "Retraining" for data engineers is really refreshing/repairing pipelines, configs, and
checks as the world changes.
- *Build consequence:* Instrument health + data signals with alerting, run a real incident process
  (detect → triage via lineage → fix → prevent), and close every incident with a new safeguard. This
  is what keeps the platform trustworthy month over month — the SLAs/SLOs of Chapter 11 made
  operational.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Describe end-to-end CI for dbt (env, seed data, tests, slim CI) to catch breaking changes pre-merge.
2. Best practices for deploying views in Databricks across dev/prod without dev-specific schema?
3. How do you handle retries and failures in ETL pipelines?
4. How do you monitor a workflow (e.g. in Airflow)?
5. How can big data achieve high observability?
6. Describe an AWS architecture using S3 and CloudWatch monitoring.
7. How do you ensure data integrity when translating T-SQL transformations into PySpark?
8. How would you containerize an environment to run and test pipeline code?
9. How do you define, measure, and report SLAs/SLOs for a pipeline?
10. How do you manage environments and promotion from dev to prod?
11. What tests do you write for a data pipeline (unit/data/integration)?
12. How do you version-control and deploy transformations and infrastructure?
13. How do you set up alerting and an incident process for pipeline failures?
14. What does "retraining"/refresh mean for a data platform you operate?
15. How do you prevent a fixed data bug from silently recurring?

_(DataOps/CI-CD/testing terms appear ~70 times in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** For the capstone repo, add the operational layer: define dev vs prod targets
(parameterized, no hard-coded schema), a CI config that runs the dbt/data tests on a pull request,
and a one-page runbook — the golden signals you'd monitor, the alert thresholds, and the
detect→triage→fix→prevent steps for one realistic incident.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~3–4 hours.

---
