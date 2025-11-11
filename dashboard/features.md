# Internal Data Tooling Dashboard (for exploring legacy data and building transformations)

- **Source discovery & connectors**
  - Mainframe/legacy (DB2, IMS, AS/400), RDBMS (Oracle, SQL Server, Postgres), files (CSV/Parquet/Avro/Excel/fixed‑width), EBCDIC, COBOL copybooks, S3/GCS/Azure, Kafka, REST/SOAP.
  - Schema inference, encoding detection, timezone/currency normalization, incremental/CDC setup.

- **Catalog, metadata & glossary**
  - Dataset/table/column registry, ownership, SLAs, tags, domain links.
  - Business glossary terms mapped to fields; data dictionary import/export.
  - Schema history, diffs, deprecation notices, contract status.

- **Profiling & data quality**
  - Column stats, distributions, nulls, uniqueness, outliers, drift.
  - Rule framework (Great Expectations/dbt tests): constraints, freshness, referential integrity.
  - Quality SLAs, failure impact, auto-ticketing.

- **Lineage & impact analysis**
  - End‑to‑end lineage graph (source → transform → models → dashboards/APIs).
  - Field‑level lineage where available; change impact preview for schema updates.

- **Exploration workspace**
  - SQL editor with cross‑source federation, visual query builder, sample & time‑travel views.
  - Versioned notebooks (Python/SQL/R) with environment packs and data access policies.
  - Saved queries, snippets, explain plan, result caching.

- **Modeling & semantics**
  - Canonical data model designer; mapping from legacy fields to canonical entities.
  - Data contracts (schemas, constraints, SLAs) and contract tests.
  - Semantic layer (dimensions/measures), metric definitions, reuse across tools.

- **Transformations & pipelines**
  - Visual DAG + code (dbt/Spark/Dagster/Airflow) with templated patterns.
  - UDF/UDTF registry, code reviews, staging → prod promotions, change approvals.
  - Dry‑runs with sampled data, diff of outputs, backfill planners with checkpoints.

- **Streaming & CDC**
  - Connect/decode (Debezium/Kafka), schema registry, replay windows.
  - Stream ↔ batch bridges, late data handling, watermarking, dedupe strategies.

- **Migration, backfills & reconciliation**
  - Idempotent backfills, throttling, partitioned runs, resumability.
  - Cross‑source reconciliation (counts, sums, tolerances), exception queues, match‑rate analytics.

- **MDM & entity resolution**
  - Matching/merging rules, survivorship policies, golden records, confidence scoring.
  - Pairs review UI and feedback loops.

- **Security, privacy & governance**
  - PII detection/classification, masking/tokenization, row/column‑level security.
  - Data access requests with time‑bound approvals; audit trails and lineage‑backed evidence.
  - Retention policies, right‑to‑erasure workflows, residency/sovereignty controls.

- **Observability & reliability**
  - Pipeline run health, freshness, SLA burn, data downtime alerts.
  - Cost/perf insights: query plans, compute/storage spend, optimization suggestions.
  - Incident timelines, runbooks, auto‑rollback/skip logic.

- **Sandboxes & test data**
  - Ephemeral environments per branch/PR with masked, sampled data.
  - Synthetic data generation, seed sets, reproducible fixtures.

- **Versioning & governance of data**
  - Dataset versioning/time‑travel (e.g., lakeFS/Delta/Iceberg), snapshot diffs.
  - Promotion gates based on tests, quality scores, contract checks.

- **Extensibility**
  - Plugin SDK (UI/backend), webhooks, events on dataset/contract changes.
  - APIs/GraphQL for metadata, lineage, quality, and execution control.

- **AI‑assisted workflows**
  - Column/metric naming suggestions, SQL generation, rule drafting.
  - Mapping suggestions from legacy to canonical, lineage explanations, root‑cause hints.

- **Non‑functional**
  - On‑prem/air‑gapped, zero‑egress connectors; encryption, HSM/KMS, authZ (RBAC/ABAC).
  - HA, backups, DR RPO/RTO; performance at TB‑scale, low‑latency lineage/search.

### Fast‑track MVP (prioritized)

1. Connectors + schema discovery for top 3 sources.
2. Profiling + basic data quality rules and freshness SLAs.
3. SQL workspace + notebooks with governed access and PII masking.
4. Lineage (table‑level) + catalog/glossary.
5. dbt/Spark integration for transforms with dry‑run/preview.
6. Backfill orchestration + reconciliation reports.
7. Contract checks in CI/CD + promotion gates.

If you share your current stack (sources, orchestrator, warehouse/lake, governance), I can map these to concrete tools and an implementation plan.
