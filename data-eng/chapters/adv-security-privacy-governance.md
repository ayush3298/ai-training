## Advanced Track — Security, Privacy & Governance

**Goal:** Learn to protect data and prove control over it — access control, PII/PHI handling and
masking, encryption, lineage and cataloging, and the regulatory frame (GDPR/HIPAA) — as engineering
you build in, not paperwork you bolt on.

**What we assume you know:** Chapters 1–11, especially Unity Catalog (Chapter 5), the medallion
boundaries, and the security/data-management undercurrents (Chapter 1).

**Why this matters:** Governance/security questions appear ~80 times in the bank, often in healthcare/
finance contexts (PII/PHI, masking, lineage, Purview). Mishandling personal data is a legal and
trust failure, and "how do you govern fact/dim tables" is a routine senior question.

> **Setup assumed:** conceptual; the mechanisms live in the platform (Unity Catalog, cloud key
> management, masking policies). Bring the capstone's data model to decide what's sensitive.

---

#### Core concepts

**1. Access control: least privilege, by role, at the right granularity.**
The base of security: people and services should access *only* what they need. In practice that's
**RBAC** (role-based access control) — grant roles, not individuals, access to catalogs/schemas/
tables, and down to **column** and **row** level (a regional analyst sees only their region's rows).
On the default stack this lives in **Unity Catalog** (Chapter 5); warehouses have equivalent grant
systems.
- *Build consequence:* Model access as roles with least privilege and use column/row-level controls
  for sensitive fields, rather than copying data into per-team tables. "How do you govern fact/dim
  tables" = catalog grants + row/column policies, centrally, not ad-hoc file permissions.

**2. PII/PHI: identify it, minimize it, and mask it — and know *when* masking happens.**
**PII** (personally identifiable information — name, email, SSN) and **PHI** (protected health
information, under HIPAA) need special handling:
  - **Identify & classify** — tag which columns are sensitive (catalog/Purview classification).
  - **Minimize** — don't ingest/keep sensitive fields you don't need.
  - **Mask / tokenize / hash** — show `j***@x.com` or a hashed token instead of the raw value to
    users who shouldn't see it. The bank's "does masking happen during processing or at storage?"
    distinguishes **dynamic masking** (data stored raw, masked *at query time* per role) from
    **static masking / tokenization** (the raw value is transformed *before/at* storage so it's never
    persisted in the clear).
- *Build consequence:* Classify sensitive columns, collect the minimum, and choose masking by need:
  dynamic masking when some roles legitimately need the raw value, static/tokenization when the raw
  value should never land. In healthcare (PHI), default to stronger handling and document it.

**3. Encryption: at rest and in transit, with managed keys.**
Two always-on protections: **encryption at rest** (stored bytes are encrypted on disk) and
**in transit** (TLS on every connection). Cloud platforms do at-rest by default; you manage **keys**
(cloud KMS, or customer-managed keys for stricter control) and ensure connections use TLS. dbt/Spark
connecting to sources/warehouses should use encrypted connections and pull secrets from a vault, never
hard-code them (Chapter 1's software-engineering undercurrent).
- *Build consequence:* Confirm at-rest encryption and TLS everywhere, manage keys via KMS, and source
  credentials from a secret manager — not config files or notebooks. Encryption is table stakes;
  getting *key management* and *secret hygiene* right is the engineering part.

**4. Lineage and cataloging: prove where every number came from.**
Governance needs a map: a **catalog** (Unity Catalog, Azure Purview, a data catalog) inventories
datasets with owners, classifications, and descriptions; **lineage** tracks how data flows
source → bronze → silver → gold → dashboard (Chapter 8's dbt lineage, Chapter 5's Unity Catalog). This
answers "where did this number come from?", "what's downstream if I change this column?", and "which
tables contain PII?".
- *Build consequence:* Maintain a catalog with ownership/classification and automated lineage so
  impact analysis and audits are a query, not an archaeology project. Lineage is also your incident
  blast-radius tool (Chapter 11) and your PII-discovery tool — it pays off well beyond compliance.

**5. Auditing and the right to be forgotten: data deletion is a pipeline feature.**
Regulations require you to *prove* who accessed what (**audit logs**) and to honor deletion requests
(**GDPR right-to-be-forgotten**). The hard part: a person's data is scattered across bronze, silver,
gold, and backups. Delta/lakehouse `DELETE` + `VACUUM` (Chapter 5) physically removes it; you need a
process that fans the deletion across every layer and store.
- *Build consequence:* Design for deletion up front — keep a way to find all of one subject's rows
  (a stable key, classified PII) and a process to delete across layers — so a "forget me" request is
  a runnable job, not a frantic manual hunt. Keep audit logs of access for the same reason.

**6. Regulation as engineering: GDPR, HIPAA, and governance-by-design.**
The major frames: **GDPR** (EU personal-data rights — consent, access, deletion, residency),
**HIPAA** (US health data — PHI safeguards, de-identification, BAAs). You don't need to be a lawyer,
but you translate requirements into controls: classification, masking, access control, audit,
residency (keep data in-region), and retention limits. Governance is *engineered* — versioned
policies, automated checks, documented data flows.
- *Build consequence:* Turn each regulatory requirement into a concrete control you build (residency →
  region-pinned storage; deletion → the fan-out job; minimization → drop unneeded PII; access → RBAC +
  audit). "Improve data governance" means adding these controls and showing their impact (e.g.
  classification + masking cut exposure), exactly as the bank asks.

---

#### Interview drill (self-test)

Real questions from the bank, scoped to this chapter.

1. Explain a strategy you used to improve data governance and its impact on data quality.
2. How did you handle PII and PHI data masking in healthcare?
3. Does masking happen during processing or at storage — and what's the difference?
4. How do you handle data governance in fact and dimension tables?
5. How do you maintain data lineage?
6. How did you implement lineage classification in Microsoft Purview?
7. How do you manage encryption and data in transit (e.g. in GCP)?
8. How do you handle encryption and decryption in dbt?
9. How did you manage PII, masking, and data sensitivity on a project?
10. Have you done migrations ensuring lineage tracking (e.g. to Microsoft Fabric)?
11. How would you implement row-level and column-level access control?
12. How would you honor a GDPR deletion request across a lakehouse?
13. What audit/access logging would you put on sensitive tables?
14. How do you keep secrets/credentials out of code in pipelines?
15. What controls map to GDPR vs HIPAA requirements?

_(governance/security/PII terms appear ~80 times in the bank; 15 shown — more in
[data-eng-questions.md](../data-eng-questions.md).)_

---

**Deliverable:** For the capstone data model, produce a one-page governance note: classify each column
(PII/PHI/none), choose dynamic vs static masking per sensitive field with a reason, define the RBAC
roles and any row/column policies, sketch the GDPR-deletion fan-out across layers, and list the
controls you'd map to GDPR/HIPAA.

**Daily update:** one-liner — done / where you stopped / any blockers.

**Time:** ~3–4 hours.

---
