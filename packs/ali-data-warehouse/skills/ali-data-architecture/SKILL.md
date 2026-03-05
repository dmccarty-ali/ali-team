---
name: ali-data-architecture
description: |
  Data architecture patterns for warehouses and transactional systems. Use when:

  PLANNING: Designing data models, planning ETL/ELT pipelines, architecting
  medallion layers (LAKE/STG/EDW), considering entity resolution approaches

  IMPLEMENTATION: Building data pipelines, implementing SCD patterns, writing
  MERGE statements, creating fact/dimension tables, implementing data quality

  GUIDANCE: Asking about data modeling best practices, medallion architecture,
  when to use ETL vs ELT, how to handle slowly changing dimensions

  REVIEW: Reviewing data models, checking pipeline idempotency, validating
  data quality rules, auditing entity resolution logic

  Do NOT use for statistical analysis, hypothesis testing, or EDA workflows (use
  ali-data-science instead), or predictive modeling and ML pipelines (use ali-machine-learning instead)
---

# Data Architecture

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing data warehouse or data lake architecture
- Planning ETL/ELT pipeline structure
- Choosing between data modeling approaches
- Architecting entity resolution systems

**Implementation:**
- Building medallion architecture layers (LAKE/STG/EDW)
- Implementing slowly changing dimensions
- Writing incremental load patterns (MERGE, CDC)
- Creating fact and dimension tables

**Guidance/Best Practices:**
- Asking about star schema vs snowflake
- Asking when to transform (ETL vs ELT)
- Asking how to handle data quality
- Asking about partitioning strategies

**Review/Validation:**
- Reviewing data models for correctness
- Checking pipeline idempotency
- Validating data quality rules
- Auditing lineage and reconciliation

---

## Key Principles

- **Single source of truth**: One authoritative location for each data element
- **Idempotency**: Pipelines can be re-run safely without duplicates
- **Immutable raw layer**: Never modify source data; transform into new layers
- **Schema on read vs write**: Raw = flexible schema; curated = strict schema
- **Data quality at boundaries**: Validate at ingestion, not downstream
- **Lineage tracking**: Know where every value came from
- **Grain clarity**: Every table has a clearly defined grain (one row = what?)

---

## Medallion Architecture

### Layer Overview

| Layer | Aliunde Name | Purpose | Data State |
|-------|--------------|---------|------------|
| Bronze | LAKE | Raw ingested data | As-received, append-only |
| Silver | STG | Cleaned, validated | Deduplicated, typed, validated |
| Gold | EDW | Business-ready | Aggregated, joined, enriched |

For complete layer schema examples with all columns and indexes, see **references/medallion-architecture.md** covering:
- LAKE layer schema (raw, append-only, immutable)
- STG layer schema (cleaned, typed, audit columns, SCD Type 2 ready)
- EDW layer schema (star schema, surrogate keys, pre-aggregated)
- Loading patterns (LAKE → STG → EDW)
- Performance optimization (clustering, materialized views)

---

## Star Schema Design

### Overview

Facts contain measurements at a specific grain. Dimensions describe context (who, what, where, when).

**Fact tables:**
- Surrogate keys (FK to dimensions)
- Degenerate dimensions (no separate table needed)
- Measures: additive (sum), semi-additive (sum across some dimensions), non-additive (ratios)

**Dimension tables:**
- Surrogate key (primary key)
- Natural key (business identifier)
- Descriptive attributes
- SCD Type 2 columns (valid_from, valid_to, is_current)

### Grain Definition

| Table | Grain (One Row Per) |
|-------|---------------------|
| FACT_CLAIM | One claim |
| FACT_CLAIM_LINE | One service line within a claim |
| FACT_PAYMENT | One payment transaction |
| FACT_DAILY_SUMMARY | One provider + date combination |

For complete fact/dimension table schemas with all columns, see **references/medallion-architecture.md** (EDW Layer section).

---

## Slowly Changing Dimensions (SCD)

Three patterns for handling dimension changes:

- **Type 1**: Overwrite (no history) - Use for corrections, non-historical attributes
- **Type 2**: Add new row (full history with date ranges) - Use when history matters
- **Type 3**: Add column (current + previous only) - Use for simple before/after tracking

For complete SQL implementations of all three SCD types with MERGE patterns and querying examples, see **references/scd-patterns.md** covering:
- Type 1: Simple UPDATE and MERGE patterns
- Type 2: Full history with valid_from/valid_to/is_current
- Type 3: Additional columns for previous values
- Hybrid approaches (different strategies per attribute)
- Mini-dimensions for high-cardinality optimization

---

## Entity Resolution

Match records across systems despite variations (typos, formatting).

**Approaches:**
- **Deterministic**: Exact match on standardized keys (fast, high precision)
- **Probabilistic**: Fuzzy matching with confidence scoring (handles variations)
- **Blocking**: Reduce comparison space with candidate generation (essential for scale)

For complete matching implementations, see **references/entity-resolution.md** covering:
- Deterministic matching with standardized keys
- Probabilistic matching with scoring (Python RapidFuzz examples)
- Blocking strategies (phonetic, multi-block)
- Machine learning approaches with feature engineering
- Match workflows (preprocessing, clustering, master record selection)
- Performance optimization (indexing, parallel processing)

---

## Data Quality

### Validation Categories

1. **Required fields**: Ensure presence of mandatory data
2. **Format validation**: Regex patterns, length checks
3. **Range validation**: Numeric bounds, date ranges
4. **Referential integrity**: Foreign key existence
5. **Business rules**: Cross-field logic (service_date < submission_date)
6. **Data quality flags**: Warnings for recommended fields

### Reconciliation Checkpoints

- **Row count**: Compare counts across layers (LAKE → STG → EDW)
- **Amount totals**: Financial reconciliation (sum of charges, payments)
- **Claim-level**: Identify records missing in downstream layers

For complete validation implementations and reconciliation SQL, see **references/data-quality.md** covering:
- Python validation framework with ValidationResult class
- SQL constraint-based validation
- Reconciliation queries (row counts, amounts, claim-level)
- Data profiling statistics
- Anomaly detection (outliers, duplicates)

---

## ETL vs ELT

| Approach | When to Use |
|----------|-------------|
| **ETL** (Extract-Transform-Load) | Limited target compute; complex transformations; data cleansing before load |
| **ELT** (Extract-Load-Transform) | Powerful target (Snowflake); raw data lake; transform flexibility |

### Our Pattern: ELT

```
Source → Stage (S3) → COPY INTO (LAKE) → Transform (SQL) → STG → EDW
         ↑ minimal processing              ↑ all transformation here
```

---

## Incremental Loading

### Watermark Pattern

Track high-water mark (timestamp, ID) for incremental extracts.

```sql
-- Control table tracks last extracted value
CREATE TABLE SYS_CONTROL.LOAD_WATERMARKS (
    table_name          VARCHAR(100) PRIMARY KEY,
    watermark_column    VARCHAR(100) NOT NULL,
    last_load_value     VARCHAR(200),
    last_load_timestamp TIMESTAMP_NTZ
);

-- Extract incremental data
SELECT * FROM source_table
WHERE updated_at > (
    SELECT last_load_value FROM SYS_CONTROL.LOAD_WATERMARKS
    WHERE table_name = 'source_table'
);
```

### MERGE Pattern (Upsert)

Insert new records, update existing records.

```sql
MERGE INTO STG_MEDEDI.CLAIM tgt
USING staging_claims src
ON tgt.claim_id = src.claim_id
WHEN MATCHED THEN UPDATE SET
    tgt.total_charge = src.total_charge,
    tgt.status = src.status
WHEN NOT MATCHED THEN INSERT
    (claim_id, total_charge, status)
    VALUES (src.claim_id, src.total_charge, src.status);
```

For complete incremental loading patterns, see **references/incremental-loading.md** covering:
- Watermark control tables with lookback windows
- MERGE patterns (basic, with deduplication, conditional updates, soft deletes)
- DELETE + INSERT pattern for small batches
- Hash-based change detection (avoid updating unchanged rows)
- Snowflake Streams (native CDC)
- Performance optimization (partitioning, clustering, parallel MERGE)

---

## Idempotency Patterns

### Delete-and-Reload

```sql
-- Safe for small batches
BEGIN TRANSACTION;
DELETE FROM STG_MEDEDI.CLAIM WHERE load_date = :batch_date;
INSERT INTO STG_MEDEDI.CLAIM SELECT * FROM staging_claims WHERE load_date = :batch_date;
COMMIT;
```

### Hash-Based Deduplication

```sql
-- Compute hash of business key + content
INSERT INTO STG_MEDEDI.CLAIM
SELECT
    MD5(claim_id || '|' || provider_npi || '|' || service_date) AS claim_key,
    *
FROM staging_claims src
WHERE NOT EXISTS (
    SELECT 1 FROM STG_MEDEDI.CLAIM tgt
    WHERE tgt.claim_key = MD5(src.claim_id || '|' || src.provider_npi || '|' || src.service_date)
);
```

---

## Window Functions

Essential for deduplication, running totals, rankings, and period-over-period comparisons.

```sql
-- Deduplication: Keep latest version
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY claim_id ORDER BY updated_at DESC) AS rn
    FROM claims
) WHERE rn = 1;

-- Running total
SELECT
    client_id, payment_date, amount,
    SUM(amount) OVER (PARTITION BY client_id ORDER BY payment_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM payments;

-- Year-over-year comparison
SELECT
    client_id, tax_year, refund_amount,
    LAG(refund_amount) OVER (PARTITION BY client_id ORDER BY tax_year) AS prior_year_refund,
    refund_amount - LAG(refund_amount) OVER (PARTITION BY client_id ORDER BY tax_year) AS yoy_change
FROM tax_returns;
```

For complete window function examples, see **references/window-functions.md** covering:
- ROW_NUMBER, RANK, DENSE_RANK (deduplication, ranking)
- SUM, AVG (running totals, moving averages)
- LAG, LEAD (period-over-period comparisons)
- FIRST_VALUE, LAST_VALUE, NTH_VALUE (boundary values)
- Complex patterns (gap/island detection, session identification)
- Frame specifications (ROWS vs RANGE)
- Performance optimization

---

## Change Data Capture (CDC)

### Approaches

- **Custom CDC**: Add created_at, updated_at, is_deleted columns; extract based on timestamps
- **Snowflake Streams**: Native CDC with METADATA$ columns; automatically tracks INSERT/UPDATE/DELETE

For complete CDC implementations, see **references/cdc-patterns.md** covering:
- Custom CDC with tracking columns and stored procedures
- Snowflake Streams with MERGE processing
- Change table pattern with before/after values

---

## Advanced Patterns

### Temporal Tables

Track data as of any point in time with bitemporal design (valid time + transaction time).

See **references/temporal-tables.md** for complete schema and querying patterns.

### Surrogate Key Generation

Methods: Sequence-based (auto-increment), Hash-based (deterministic), UUID (globally unique).

See **references/surrogate-keys.md** for complete implementations and trade-offs.

### Data Mesh

Federated data ownership with domain data products, self-serve platforms, and global governance.

See **references/data-mesh.md** for principles, data product definitions, and benefits/challenges.

### Partitioning

Table partitioning and clustering for query performance.

See **references/partitioning-strategies.md** for strategies (time, hash, list) and Snowflake clustering.

### Lineage

Column-level lineage tracking for compliance and debugging.

See **references/lineage.md** for YAML lineage definitions and standard audit columns.

---

## Naming Conventions

### Quick Reference

**Standard abbreviations**: ABBR, ACCT, ACK, ADDL, ADDR, ADJ, AMT, AUTH, CHRG, CLM, DESC, etc.

**Table prefixes**: LU_ (lookup), XREF_ (cross-reference), DIM_ (dimension), FACT_ (fact), STG_ (staging), LAKE_ (raw), EDW_ (warehouse)

**Column suffixes**: _SK (surrogate key), _ID (natural ID), _CODE (code value), _DESC (description), _FLAG (boolean), _TS (timestamp), _DATE (date), _AMT (amount), _QTY (quantity)

For complete abbreviation table (130+ entries), prefixes, suffixes, and naming best practices, see **references/naming-conventions.md**.

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Table definitions | `metadata/tbl_defs.yaml` |
| Input mappings | `metadata/input_map.yaml` |
| SQL generators | `sql_gen_build/` |
| Generated DDL | `sql_gen_build/output/` |
| Data flow SQL | `sql_gen_build/sql_gen_data_flow.py` |
| Match rules | `sql_gen_build/sql_gen_match_rules.py` |

---

## Do NOT Use For

- **Database-specific SQL optimization** (use ali-snowflake-core or ali-database-developer instead)
- **Snowflake ingestion pipelines with Snowpipe/Kafka** (use ali-snowflake-data-engineer instead)
- **Apache Iceberg table format configuration** (use ali-apache-iceberg instead)
- **OLTP schema design with migrations** (use ali-database-developer instead)
- **Application ORM models (SQLAlchemy, Prisma)** (use ali-backend-developer instead)

---

## References

- [Kimball Dimensional Modeling](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/)
- [Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
- [Data Vault 2.0](https://datavaultalliance.com/data-vault-2-0-overview/)
- [Snowflake Best Practices](https://docs.snowflake.com/en/user-guide/performance-overview)

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Refactored:** Progressive disclosure pattern with references/ subdirectory
