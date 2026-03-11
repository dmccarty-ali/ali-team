---
name: ali-snowflake-data-engineer
description: |
  Advanced Snowflake data engineering aligned with SnowPro Advanced Data Engineer
  (DEA-C02). Use when:

  PLANNING: Designing real-time ingestion pipelines, planning Snowpipe Streaming
  architecture, architecting medallion layers (Bronze/Silver/Gold), choosing
  Iceberg table strategies, query optimization strategies

  IMPLEMENTATION: Building Kafka connectors, writing Snowpark transformations,
  implementing lock-free staging patterns, creating QUALIFY deduplication logic,
  building hash-based MERGE change detection, dynamic warehouse parallelism

  GUIDANCE: Asking about Snowpipe vs Snowpipe Streaming tradeoffs, dynamic warehouse
  parallelism, row-level security patterns, external table usage, data lineage tracking,
  performance optimization best practices

  REVIEW: Validating streaming ingestion configs, reviewing Snowpark DataFrame logic,
  checking task DAG dependencies, auditing data quality monitoring, verifying
  replication/failover setup, performance bottleneck analysis

  Do NOT use for basic Snowflake concepts like warehouse sizing, Time Travel,
  micro-partitions basics, or simple COPY INTO statements (use ali-snowflake-core
  instead). Do NOT use for account administration, user management, or cost monitoring
  (use ali-snowflake-admin instead).
---

# Snowflake Data Engineer - Advanced Topics

## Overview

This skill provides advanced Snowflake knowledge for data engineers, organized around the five domains of the SnowPro Advanced: Data Engineer Certification (DEA-C02). It builds on ali-snowflake-core fundamentals with deep dives into pipeline design, optimization, and governance.

**Prerequisites:** Familiarity with ali-snowflake-core concepts

**Certification Domain Weightings:**
| Domain | Weight |
|--------|--------|
| 1. Data Movement | 26% |
| 2. Performance Optimization | 21% |
| 3. Storage & Data Protection | 14% |
| 4. Data Governance | 14% |
| 5. Data Transformation | 25% |

---

## Negative Triggers - When NOT to Use This Skill

**Do NOT use for:**
- Basic Snowflake concepts like warehouse sizing, Time Travel, or micro-partitions basics (use ali-snowflake-core instead)
- Account-level administration like user management, role hierarchies, or cost monitoring (use ali-snowflake-admin instead)
- Simple COPY INTO statements or basic table creation (use ali-snowflake-core instead)
- General SQL queries without data engineering patterns (use ali-snowflake-core instead)
- Account setup, network policies, or billing (use ali-snowflake-admin instead)

**DO use for:**
- Real-time ingestion architecture (Snowpipe Streaming, Kafka connectors)
- Advanced performance optimization (clustering analysis, query acceleration)
- Lock-free staging patterns for parallel processing
- QUALIFY-based deduplication
- Hash-based MERGE change detection
- Medallion architecture (Bronze/Silver/Gold)
- Snowpark transformations
- Java JDBC integration patterns
- Dynamic warehouse parallelism
- Advanced governance (tagging, masking, row access policies)

---

## Domain 1: Data Movement (26%)

### When to Use Each Ingestion Method

| Scenario | Use Snowpipe | Use Streaming |
|----------|--------------|---------------|
| File-based sources (S3, GCS) | Yes | No |
| Kafka/event streams | Maybe | Yes |
| Sub-second latency needed | No | Yes |
| High-volume continuous | Yes | Yes |
| Simple setup | Yes | No (requires SDK) |

**For detailed implementation:**
- Snowpipe Streaming API code: See references/snowpipe-streaming-api.md covering:
  - Complete Java SDK implementation
  - Kafka connector configuration
  - Spark connector patterns
- Iceberg tables: See references/iceberg-tables.md covering DDL and external volume setup
- Data sharing & replication: See references/data-sharing-replication.md covering:
  - Row-level filtering for shares
  - Cross-cloud/cross-region replication
  - Failover/failback procedures
- Troubleshooting: See references/troubleshooting.md covering:
  - Common COPY INTO errors and resolutions
  - Snowpipe troubleshooting queries
  - Performance analysis SQL

### Kafka Connector Ingestion Method Selection

**Ingestion Methods:**
- **SNOWPIPE:** Stages files, then loads (minute latency)
- **SNOWPIPE_STREAMING:** Direct row insert (sub-second latency)

---

## Domain 2: Performance Optimization (21%)

### Query Profile Key Operators

| Operator | Warning Signs | Solutions |
|----------|---------------|-----------|
| TableScan | High partitions scanned | Add clustering, improve filters |
| Join | Spillage, explosion | Filter earlier, check join keys |
| Sort | Spillage | Larger warehouse, limit results |
| Aggregate | Spillage | Pre-aggregate, sample first |
| WindowFunction | High memory | Partition wisely, limit window |

**For detailed analysis:**
- Query troubleshooting SQL: See references/troubleshooting.md covering top slow queries and partition pruning analysis
- Query acceleration: See references/query-acceleration.md covering setup and cost monitoring
- Pipeline monitoring: See references/pipeline-monitoring.md covering:
  - Stream health monitoring
  - Task monitoring and failure detection
  - Alert configuration

### Clustering Key Selection Guidance

**Good clustering keys:**
1. Columns in WHERE clauses
2. Columns in JOIN conditions
3. Date/timestamp columns (most queries filter by time)
4. Low-to-medium cardinality columns

**Examples:**
```sql
-- Multi-column clustering
ALTER TABLE events CLUSTER BY (event_date, event_type);

-- Expression-based clustering
ALTER TABLE events CLUSTER BY (DATE_TRUNC('DAY', event_timestamp), region);
```

**For detailed analysis:** See references/clustering-analysis.md covering SYSTEM$CLUSTERING_INFORMATION JSON output and reclustering commands.

### Search Optimization Best Practices

```sql
-- Enable search optimization (table level)
ALTER TABLE my_table ADD SEARCH OPTIMIZATION;

-- Enable for specific columns/operations
ALTER TABLE my_table ADD SEARCH OPTIMIZATION
  ON EQUALITY(customer_id, order_id)
  ON SUBSTRING(description)
  ON GEO(location);
```

**Best for:**
- Point lookups: WHERE id = 'abc123'
- IN lists: WHERE id IN (1, 2, 3, ...)
- Substring: WHERE desc LIKE '%keyword%'
- Geospatial: WHERE ST_CONTAINS(...)

---

## Domain 3: Storage & Data Protection (14%)

**For detailed analysis:**
- Storage analysis: See references/storage-analysis.md covering:
  - Table storage metrics queries
  - Advanced cloning patterns
  - Clone lineage tracking

---

## Domain 4: Data Governance (14%)

**For comprehensive governance patterns:**
- See references/governance.md covering:
  - Object tagging strategy and queries
  - Dynamic data masking policies
  - Row access policies (multi-tenant, region-based)
  - Data lineage tracking
  - Data quality monitoring with alerts

---

## Domain 5: Data Transformation (25%)

**For detailed transformation patterns:**
- Advanced UDFs: See references/advanced-udfs.md covering:
  - Vectorized Python UDFs
  - UDTFs (Table Functions)
  - External Functions (API calls)
- Snowpark API: See references/snowpark-api.md covering:
  - DataFrame API transformations
  - Snowpark stored procedures
- Semi-structured data: See references/semi-structured.md covering:
  - Deep JSON extraction
  - Recursive flattening
  - JSON aggregation
- Unstructured data: See references/unstructured-data.md covering:
  - Directory tables
  - Pre-signed URLs
  - PDF extraction UDFs
- Stream + Task CDC: See references/stream-task-cdc.md covering complete task DAG implementation

---

## Pipeline Design Patterns

### Lock-Free Staging Pattern

**Problem:** Multiple parsers writing INSERT/UPDATE to same table causes lock contention and timeouts.

**Solution:**
1. Parallel processes INSERT to staging table (no locks on INSERT)
2. Single MERGE consolidates staging to target after all processes complete

```sql
-- Staging table: receives parallel INSERTs from multiple parsers
CREATE TABLE SYS_CONTROL.FILES_STATUS_STAGING (
    batch_id            VARCHAR(100),
    parser_seq          INTEGER,          -- Which parser instance
    file_key            VARCHAR(500),
    status              VARCHAR(50),
    record_count        INTEGER,
    error_message       VARCHAR(4000),
    created_at          TIMESTAMP_NTZ(6), -- Microsecond precision
    PRIMARY KEY (batch_id, parser_seq, file_key)
);

-- After all parsers complete: single MERGE consolidates to catalog
MERGE INTO SYS_CONTROL.FILES_CATALOG tgt
USING (
    -- Get latest status per file using QUALIFY
    SELECT file_key, status, record_count, created_at
    FROM SYS_CONTROL.FILES_STATUS_STAGING
    WHERE batch_id = :batch_id
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY file_key
        ORDER BY created_at DESC
    ) = 1
) src
ON tgt.file_key = src.file_key
WHEN MATCHED THEN UPDATE SET
    status = src.status,
    record_count = src.record_count,
    last_updated = src.created_at
WHEN NOT MATCHED THEN INSERT (file_key, status, record_count, last_updated)
VALUES (src.file_key, src.status, src.record_count, src.created_at);
```

**Key insight:** INSERT operations don't lock, MERGE at the end consolidates atomically.

### QUALIFY Deduplication Pattern - Basics

QUALIFY filters window function results without subquery:

```sql
-- Get latest record per key based on timestamp
SELECT *
FROM events
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY event_id
    ORDER BY updated_at DESC
) = 1;

-- Equivalent subquery (more verbose)
SELECT * FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY updated_at DESC) as rn
    FROM events
) WHERE rn = 1;
```

**For advanced patterns:** See references/qualify-advanced.md covering microsecond precision and multi-column deduplication.

### Hash-Based Change Detection in MERGE

Detect actual data changes to avoid unnecessary updates:

```sql
-- Generate row hash for change detection
CREATE VIEW STG_MEDEDI.CLAIM_WITH_HASH AS
SELECT
    claim_id,
    patient_id,
    provider_id,
    charge_amount,
    -- Hash of all non-key columns for change detection
    HASH(patient_id, provider_id, charge_amount, service_date, diagnosis_code) AS row_hash
FROM STG_MEDEDI.CLAIM;

-- MERGE with hash comparison (only update if data changed)
MERGE INTO EDW_MEDEDI.FACT_CLAIM tgt
USING STG_MEDEDI.CLAIM_WITH_HASH src
ON tgt.claim_id = src.claim_id
WHEN MATCHED AND tgt.row_hash != src.row_hash THEN UPDATE SET
    patient_id = src.patient_id,
    provider_id = src.provider_id,
    charge_amount = src.charge_amount,
    row_hash = src.row_hash,
    updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN INSERT (
    claim_id, patient_id, provider_id, charge_amount, row_hash, created_at
) VALUES (
    src.claim_id, src.patient_id, src.provider_id, src.charge_amount,
    src.row_hash, CURRENT_TIMESTAMP()
);
```

**Key insight:** HASH() returns NUMBER(19,0), deterministic, suitable for surrogate keys and change detection.

### Medallion Architecture Overview

```sql
-- Bronze: Raw ingestion
CREATE SCHEMA bronze;
CREATE TABLE bronze.raw_events (
    ingestion_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    source_file VARCHAR,
    raw_data VARIANT
);

-- Silver: Cleaned and typed
CREATE SCHEMA silver;
CREATE TABLE silver.events (
    event_id INT,
    event_type VARCHAR,
    event_time TIMESTAMP,
    user_id INT,
    properties VARIANT,
    _bronze_id INT,
    _processed_at TIMESTAMP
);

-- Gold: Business aggregates
CREATE SCHEMA gold;
CREATE TABLE gold.daily_user_activity (
    activity_date DATE,
    user_id INT,
    event_count INT,
    unique_event_types INT,
    first_event_time TIMESTAMP,
    last_event_time TIMESTAMP
);

-- Stream + Task pipeline for each layer
CREATE STREAM bronze.events_stream ON TABLE bronze.raw_events;
CREATE STREAM silver.events_stream ON TABLE silver.events;
```

**For complete CDC implementation:** See references/stream-task-cdc.md

---

## Java JDBC Integration Patterns

**For complete Java patterns:** See references/jdbc-patterns.md covering:
- Connection configuration (password and RSA key pair auth)
- Query tag for monitoring
- PUT operations from Java
- Dynamic warehouse parallelism calculation

**Dynamic parallelism example (Python):**
```python
# Map warehouse sizes to compute nodes
WAREHOUSE_NODE_COUNT = {
    'X-SMALL': 1, 'SMALL': 2, 'MEDIUM': 4, 'LARGE': 8,
    'X-LARGE': 16, '2X-LARGE': 32, '3X-LARGE': 64, '4X-LARGE': 128
}

def get_warehouse_thread_count(conn, threads_per_node: int = 4) -> int:
    size = get_warehouse_size(conn)
    nodes = WAREHOUSE_NODE_COUNT.get(size, 1)
    return nodes * threads_per_node
```

---

## Quick Reference

### Performance Checklist

```
□ Check partition pruning (Query Profile)
□ Review clustering depth for large tables
□ Enable search optimization for point lookups
□ Size warehouse appropriately (check spillage)
□ Consider materialized views for repeated aggregations
□ Use query acceleration for ad-hoc analytics
□ Monitor stream lag and task failures
```

### Data Movement Checklist

```
□ Choose ingestion method (COPY/Snowpipe/Streaming)
□ Define file format with error handling
□ Set appropriate ON_ERROR behavior
□ Configure auto-ingest notifications
□ Monitor load history for failures
□ Plan for schema evolution
```

---

## Detailed References

For in-depth implementation details, consult the following references:

| Topic | Reference File | Coverage |
|-------|---------------|----------|
| Snowpipe Streaming API | references/snowpipe-streaming-api.md | Java SDK, Kafka connector, Spark connector |
| Iceberg Tables | references/iceberg-tables.md | DDL, external volumes, schema evolution |
| Data Sharing & Replication | references/data-sharing-replication.md | Cross-cloud, failover, row-level filtering |
| Troubleshooting | references/troubleshooting.md | COPY errors, Snowpipe issues, slow queries |
| Clustering Analysis | references/clustering-analysis.md | SYSTEM$CLUSTERING_INFORMATION, reclustering |
| Query Acceleration | references/query-acceleration.md | Setup, eligibility, cost monitoring |
| Pipeline Monitoring | references/pipeline-monitoring.md | Streams, tasks, alerts |
| Storage Analysis | references/storage-analysis.md | Storage metrics, cloning, lineage |
| Governance | references/governance.md | Tagging, masking, row access, lineage, quality |
| Advanced UDFs | references/advanced-udfs.md | Vectorized UDFs, UDTFs, external functions |
| Snowpark API | references/snowpark-api.md | DataFrame API, stored procedures |
| Semi-Structured Data | references/semi-structured.md | JSON extraction, flattening, aggregation |
| Unstructured Data | references/unstructured-data.md | Directory tables, PDF processing |
| Stream + Task CDC | references/stream-task-cdc.md | Complete task DAG implementation |
| Java JDBC Patterns | references/jdbc-patterns.md | Connection, query tags, PUT, parallelism |
| QUALIFY Advanced | references/qualify-advanced.md | Microsecond precision, multi-column dedup |

---

## Related Skills

- **ali-snowflake-core:** Foundational concepts (architecture, basic SQL, security basics)
- **ali-snowflake-admin:** Account administration, cost management, replication setup

---

## Official Resources

- [Snowflake Data Engineering Guide](https://docs.snowflake.com/en/user-guide/data-load-overview)
- [Snowpark Developer Guide](https://docs.snowflake.com/en/developer-guide/snowpark/index)
- [SnowPro Advanced: Data Engineer](https://learn.snowflake.com/en/certifications/snowpro-advanced-dataengineer-C02/)

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Based On:** SnowPro Advanced Data Engineer DEA-C02 Exam Domains
**Maintained By:** Aliunde

**Progressive Disclosure Refactoring:**
- Core decision-making content retained in SKILL.md (~350 lines)
- Detailed implementation patterns moved to references/ directory (16 files)
- Context savings: ~79% reduction in skill load size
- Reference files loaded on demand via Read tool when needed
