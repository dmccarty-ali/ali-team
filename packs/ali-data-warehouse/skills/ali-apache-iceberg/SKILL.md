---
name: ali-apache-iceberg
description: |
  Apache Iceberg table format for data lakes. Use when:

  PLANNING: Designing Bronze layer architecture, evaluating catalog types,
  planning partition evolution strategies, architecting CDC ingestion patterns

  IMPLEMENTATION: Creating Iceberg tables, implementing compaction jobs,
  writing MERGE INTO patterns, configuring time travel, setting up catalogs

  GUIDANCE: Asking about Iceberg vs Delta Lake vs Hudi, when to use
  hidden partitioning, how to handle schema evolution, catalog selection

  REVIEW: Reviewing table configurations, checking compaction strategies,
  validating partition schemes, auditing snapshot policies

  Do NOT use for:
  - General data architecture patterns (use ali-data-architecture) - medallion layers, SCD patterns, ETL/ELT design
  - Parquet file format details (use ali-apache-parquet) - low-level schema, PyArrow, row group optimization
  - Kafka streaming patterns (use ali-apache-kafka) - producer/consumer setup, topic design, consumer groups
  - Snowflake-specific operations (use ali-snowflake-data-engineer) - external tables, Snowpipe, Snowpark
  - Starburst/Trino federation (use ali-starburst) - query federation architecture, connector configuration
  - Database development (use ali-database-developer) - SQL optimization, indexing, transaction management
---

# Apache Iceberg

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Bronze layer table format architecture
- Choosing catalog types for metadata management
- Planning partition evolution and hidden partitioning
- Architecting CDC ingestion from Debezium/Kafka
- Evaluating Iceberg vs Delta Lake vs Hudi

**Implementation:**
- Creating Iceberg tables with proper configurations
- Implementing compaction and file optimization jobs
- Writing MERGE INTO upsert patterns
- Setting up time travel and snapshot management
- Configuring Spark/Flink/Trino/Starburst integrations

**Guidance/Best Practices:**
- Asking about partition transform best practices
- Asking when to use copy-on-write vs merge-on-read
- Asking about schema evolution compatibility
- Asking how to optimize query performance
- Asking about snapshot expiration policies

**Review/Validation:**
- Reviewing table configurations for optimization
- Checking compaction schedules and effectiveness
- Validating partition schemes and pruning
- Auditing snapshot growth and orphan files
- Checking catalog configuration and security

---

## Key Principles

- **Hidden partitioning**: Partition values computed from data, not stored as columns - users query naturally
- **Schema evolution is safe**: Add, drop, rename, reorder columns without breaking readers
- **Time travel via snapshots**: Every write creates immutable snapshot - query any historical version
- **Metadata layer is key**: Manifest files + manifest lists enable aggressive pruning without touching data
- **Compaction is mandatory**: Small file problem requires regular bin-packing and sorting
- **Catalog choice matters**: Hive Metastore, Glue, REST, Nessie, JDBC - each has tradeoffs
- **File format defaults to Parquet**: ORC and Avro supported, but Parquet is preferred
- **Merge-on-read for CDC**: Copy-on-write for batch, merge-on-read for streaming CDC ingestion

---

## Iceberg Table Format Architecture

### Metadata Layer

```
┌─────────────────────────────────────────────────────────────┐
│                      Catalog                                  │
│  (Hive Metastore, Glue, REST, Nessie, JDBC)                 │
│  Points to: metadata.json location                           │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              metadata.json (Current Snapshot)                 │
│  - Schema version, partition spec, sort order                │
│  - Current snapshot ID, snapshot list                        │
│  - Manifest list location                                    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Manifest List (Snapshot Metadata)                │
│  - List of manifest files                                    │
│  - Partition summaries, file counts                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Manifest Files (Data File Metadata)              │
│  - Data file locations (S3 paths)                            │
│  - Per-file statistics (min, max, null counts)               │
│  - Partition values for each data file                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Data Files (Parquet, ORC, Avro)                  │
│  - Actual table data                                         │
└─────────────────────────────────────────────────────────────┘
```

### Why This Architecture

| Layer | Purpose | Pruning Benefit |
|-------|---------|-----------------|
| **Catalog** | Single source of truth for table location | No scanning for tables |
| **metadata.json** | Current schema, snapshots, manifest list | Schema without data access |
| **Manifest List** | Partition summaries across manifest files | Prune manifest files by partition |
| **Manifest Files** | Per-file min/max statistics | Prune data files by predicate |
| **Data Files** | Row-level data | Only read surviving files |

**Example query plan optimization:**
```sql
SELECT * FROM orders WHERE order_date = '2026-01-15' AND region = 'US-WEST';

-- Pruning cascade:
1. Catalog → metadata.json (0ms)
2. Manifest list pruning: 100 manifest files → 5 files (partition match)
3. Manifest file pruning: 1000 data files → 20 files (statistics match)
4. Data file reads: 20 Parquet files (instead of 1000)
```

---

## Catalog Types

| Catalog | When to Use | Pros | Cons |
|---------|-------------|------|------|
| **Hive Metastore** | Existing Hive ecosystem | Wide compatibility, battle-tested | Single point of failure, not cloud-native |
| **AWS Glue** | AWS-native deployments | Serverless, HA, IAM integration | AWS-only, eventual consistency |
| **REST Catalog** | Cloud-agnostic SaaS | Centralized, multi-region | Network dependency, vendor lock-in |
| **Nessie** | Git-like versioning, multi-table transactions | Branches, tags, multi-table ACID | Additional service to manage |
| **JDBC Catalog** | Small deployments, testing | Simple, any JDBC database | Not HA, manual scaling |

**For detailed catalog configuration examples**, consult references/catalog-configuration.md covering:
- Hive Metastore configuration (Spark, Flink)
- AWS Glue Catalog setup with IAM
- REST Catalog configuration
- Nessie branching and git-like workflows
- JDBC Catalog setup for testing

---

## Hidden Partitioning

### Why Hidden Partitioning

Traditional partitioning exposes partition columns to users, requiring them to filter on partition columns for performance:

**Traditional (Hive-style) - BAD:**
```sql
SELECT * FROM events
WHERE year = 2026 AND month = 1 AND event_date = '2026-01-15';
--    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ user must include partition columns
```

**Iceberg Hidden Partitioning - GOOD:**
```sql
SELECT * FROM events
WHERE event_date = '2026-01-15';
--    ^^^^^^^^^^^^ Iceberg automatically prunes to year=2026/month=01 partition
```

### Partition Transform Functions

| Transform | Use Case | Example |
|-----------|----------|---------|
| **year(ts)** | Time-series data by year | `PARTITIONED BY (year(event_timestamp))` |
| **month(ts)** | Time-series data by month | `PARTITIONED BY (month(event_timestamp))` |
| **day(ts)** | Time-series data by day | `PARTITIONED BY (day(event_timestamp))` |
| **hour(ts)** | High-volume streaming data | `PARTITIONED BY (hour(event_timestamp))` |
| **bucket(N, col)** | Hash partitioning for even distribution | `PARTITIONED BY (bucket(16, customer_id))` |
| **truncate(L, col)** | Group by prefix (strings) | `PARTITIONED BY (truncate(2, country_code))` |
| **identity(col)** | Partition by exact value | `PARTITIONED BY (identity(region))` |

**For detailed hidden partitioning examples**, consult references/hidden-partitioning-examples.md covering:
- CREATE TABLE examples with various partition transforms
- Partition evolution (day → month without rewrite)
- Multi-level partitioning strategies
- Anti-patterns (over-partitioning, high cardinality)

---

## Schema Evolution

### Safe Schema Changes

| Operation | Safe? | Example |
|-----------|-------|---------|
| Add column | ✅ Yes | `ALTER TABLE t ADD COLUMN new_col STRING` |
| Drop column | ✅ Yes | `ALTER TABLE t DROP COLUMN old_col` |
| Rename column | ✅ Yes | `ALTER TABLE t RENAME COLUMN old_name TO new_name` |
| Reorder columns | ✅ Yes | `ALTER TABLE t ALTER COLUMN col FIRST` |
| Change type (widening) | ✅ Yes | `int → bigint`, `float → double` |
| Change type (narrowing) | ❌ No | `bigint → int` (breaks readers) |

**For detailed schema evolution syntax and examples**, consult references/schema-evolution-details.md covering:
- ADD COLUMN with positioning
- DROP COLUMN for nested structs
- RENAME COLUMN examples
- Type promotion (int → bigint, decimal precision increase)
- Type compatibility table
- Migration patterns for unsafe changes

---

## Time Travel and Snapshots

### Query Historical Snapshots

```sql
-- Query as of timestamp
SELECT * FROM orders
FOR SYSTEM_TIME AS OF '2026-01-15 10:30:00';

-- Query specific snapshot ID
SELECT * FROM orders
FOR SYSTEM_VERSION AS OF 1234567890;

-- Spark DataFrame API
spark.read \
    .option("snapshot-id", "1234567890") \
    .table("my_catalog.bronze_layer.orders")
```

### List Snapshots

```sql
-- Show snapshot history
SELECT * FROM my_catalog.bronze_layer.orders.snapshots
ORDER BY committed_at DESC;

-- Output:
-- committed_at          | snapshot_id  | parent_id    | operation
-- 2026-01-15 14:30:00  | 1234567890   | 1234567889   | append
-- 2026-01-15 10:00:00  | 1234567889   | 1234567888   | overwrite
```

**For advanced time travel operations**, consult references/time-travel-advanced.md covering:
- Rollback to previous snapshot (CALL system.rollback_to_snapshot)
- Cherry-pick snapshot (selective commit reapply)
- Snapshot expiration configuration
- Snapshot tagging for long-term retention
- Nessie branching and merging workflows

---

## Compaction

### Why Compaction is Mandatory

**Small file problem:**
- Streaming writes create thousands of small files
- Query planning overhead increases (listing files)
- Data file pruning less effective
- S3 API costs increase (more LIST/GET operations)

### When to Compact

| Workload | Bin-Pack Frequency | Sort Frequency | Target File Size |
|----------|-------------------|----------------|------------------|
| Streaming CDC | Hourly | Weekly | 512 MB |
| Batch ETL (nightly) | Daily | Weekly | 512 MB |
| Ad-hoc writes | Weekly | Monthly | 512 MB |

### Basic Compaction Example

```sql
-- Bin-pack small files into 512 MB files
CALL my_catalog.system.rewrite_data_files(
    table => 'bronze_layer.cdc_orders',
    strategy => 'binpack',
    options => map('target-file-size-bytes', '536870912')
);
```

**For detailed compaction operations**, consult references/compaction-operations.md covering:
- Bin-packing compaction (combine small files)
- Sort-order compaction (optimize for queries)
- Incremental compaction (partition-specific)
- Parallel compaction across partitions
- Compaction scheduling (cron examples)
- Monitoring compaction need

---

## Write Patterns

### Copy-on-Write vs Merge-on-Read

| Mode | When to Use | Write Cost | Read Cost | Use Case |
|------|-------------|------------|-----------|----------|
| **Copy-on-Write** | Batch updates | High (rewrite files) | Low (no merging) | Nightly batch jobs |
| **Merge-on-Read** | Streaming CDC | Low (append deltas) | Medium (merge at read) | Real-time CDC ingestion |

**Configure merge-on-read for CDC:**
```sql
CREATE TABLE cdc_orders (
    order_id BIGINT,
    status STRING,
    updated_at TIMESTAMP
)
USING iceberg
TBLPROPERTIES (
    'write.update.mode' = 'merge-on-read',
    'write.delete.mode' = 'merge-on-read',
    'write.merge.mode' = 'merge-on-read'
);
```

**For detailed write operation examples**, consult references/write-operations-detail.md covering:
- Append (INSERT) operations
- Overwrite (partition and full table)
- Upsert (MERGE INTO) with complex conditions
- UPDATE and DELETE operations
- Transactional writes (Nessie multi-table ACID)
- Write performance tuning (fanout, distribution mode)

---

## CDC Ingestion Pattern

### Debezium → Kafka → Iceberg

**Architecture:**
```
PostgreSQL → Debezium → Kafka (Avro) → Flink/Spark → Iceberg (Bronze)
```

**Debezium CDC message structure:**
```json
{
  "before": {"id": 123, "name": "Old Name", "status": "pending"},
  "after": {"id": 123, "name": "New Name", "status": "approved"},
  "op": "u",  // c=create, u=update, d=delete, r=read (snapshot)
  "ts_ms": 1705334400000
}
```

**For detailed CDC implementation**, consult references/cdc-implementation.md covering:
- Flink CDC ingestion (complete pipeline example)
- Spark Structured Streaming CDC
- Exactly-once semantics with checkpointing
- Multi-table CDC patterns
- Schema Registry integration
- CDC monitoring and troubleshooting

---

## Decision Guides

### Iceberg vs Delta Lake vs Hudi

| Feature | Iceberg | Delta Lake | Hudi |
|---------|---------|------------|------|
| **Hidden partitioning** | ✅ Yes | ❌ No | ❌ No |
| **Partition evolution** | ✅ Yes | ❌ No | ❌ No |
| **Schema evolution** | ✅ Best | ✅ Good | ⚠️ Limited |
| **Time travel** | ✅ Snapshots | ✅ Versions | ✅ Commits |
| **Engine support** | ✅ Spark, Flink, Trino, Presto, Athena | ⚠️ Spark-centric | ⚠️ Spark-centric |
| **Merge-on-read** | ✅ Yes | ✅ Yes | ✅ Yes (native) |
| **Catalog flexibility** | ✅ Multiple (Hive, Glue, REST, Nessie) | ⚠️ Hive Metastore | ⚠️ Hive Metastore |
| **Metadata layer** | ✅ Efficient (manifest files) | ✅ Transaction log | ✅ Timeline |
| **Community** | Apache (vendor-neutral) | Databricks | Apache |

**Recommendation:**
- **Iceberg**: Multi-engine analytics, flexible partitioning, vendor-neutral
- **Delta Lake**: Databricks ecosystem, Spark-heavy workloads
- **Hudi**: Streaming upserts, record-level updates, CDC-heavy

### When to Use Each Catalog Type

| Scenario | Recommended Catalog |
|----------|-------------------|
| Existing Hive ecosystem | Hive Metastore |
| AWS-native architecture | AWS Glue |
| Multi-cloud deployment | REST Catalog |
| Git-like branching/versioning | Nessie |
| Quick prototyping | JDBC Catalog |

### When to Use Iceberg

✅ **Use Iceberg when:**
- You need multi-engine support (Spark, Flink, Trino, Presto, Athena)
- Hidden partitioning and partition evolution are valuable
- You want vendor-neutral open table format
- Schema evolution is frequent
- CDC ingestion from Debezium/Kafka

❌ **Don't use Iceberg when:**
- You're locked into Databricks ecosystem (use Delta Lake)
- You need record-level update performance over everything (consider Hudi)
- Your data is small and doesn't justify overhead

---

## Crown Equipment Context

### Crown Governance Architecture

In the Crown data governance architecture:

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Operational DB** | PostgreSQL (Aurora) | Transactional workload |
| **CDC** | Debezium → Kafka | Real-time change capture |
| **Bronze Layer** | **Iceberg** | Raw CDC landing zone |
| **Query Engine** | Starburst (Trino) | Cross-Iceberg table analytics |

**Why Iceberg for Bronze:**
- Hidden partitioning: Analysts query naturally without knowing partition scheme
- Schema evolution: CDC schema changes don't break downstream
- Time travel: Query historical state for auditing
- Starburst native support: Trino has excellent Iceberg integration
- Snapshot isolation: Concurrent reads/writes without locking

### Recommended Configuration for Crown

```python
# Spark configuration for Debezium → Kafka → Iceberg pipeline
spark.sql.catalog.bronze = org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.bronze.catalog-impl = org.apache.iceberg.aws.glue.GlueCatalog
spark.sql.catalog.bronze.warehouse = s3://crown-bronze-layer/
spark.sql.catalog.bronze.io-impl = org.apache.iceberg.aws.s3.S3FileIO

# Table properties for CDC workload
CREATE TABLE bronze.operations.equipment_events (
    event_id BIGINT,
    equipment_id STRING,
    event_type STRING,
    event_timestamp TIMESTAMP,
    payload STRING
)
USING iceberg
PARTITIONED BY (days(event_timestamp))
TBLPROPERTIES (
    'write.format.default' = 'parquet',
    'write.parquet.compression-codec' = 'zstd',
    'write.update.mode' = 'merge-on-read',
    'write.delete.mode' = 'merge-on-read',
    'history.expire.max-snapshot-age-ms' = '604800000',  -- 7 days
    'write.target-file-size-bytes' = '536870912'  -- 512 MB
);
```

---

## Quick Reference

### Essential Table Properties

```sql
-- Performance
'write.target-file-size-bytes' = '536870912'  -- 512 MB
'write.parquet.compression-codec' = 'zstd'
'write.parquet.row-group-size-bytes' = '134217728'  -- 128 MB

-- CDC patterns
'write.update.mode' = 'merge-on-read'
'write.delete.mode' = 'merge-on-read'

-- Snapshot management
'history.expire.max-snapshot-age-ms' = '604800000'  -- 7 days
'history.expire.min-snapshots-to-keep' = '10'

-- Compaction
'commit.manifest.min-count-to-merge' = '5'
'commit.manifest-merge.enabled' = 'true'
```

### Common SQL Operations

```sql
-- Create table
CREATE TABLE t (id BIGINT, data STRING)
USING iceberg
PARTITIONED BY (days(ts));

-- Add column
ALTER TABLE t ADD COLUMN new_col STRING;

-- Rename column
ALTER TABLE t RENAME COLUMN old TO new;

-- Partition evolution
ALTER TABLE t REPLACE PARTITION FIELD days(ts) WITH months(ts);

-- Time travel
SELECT * FROM t FOR SYSTEM_TIME AS OF '2026-01-15';

-- Rollback
CALL system.rollback_to_snapshot('db.t', 1234567890);

-- Compaction
CALL system.rewrite_data_files('db.t', strategy => 'binpack');

-- Expire snapshots
CALL system.expire_snapshots('db.t', TIMESTAMP '2026-01-08');

-- Remove orphans
CALL system.remove_orphan_files('db.t', TIMESTAMP '2026-01-08');
```

---

## Detailed Reference Files

For in-depth examples and advanced configurations, consult the references directory:

| Reference File | Topics Covered |
|----------------|----------------|
| **catalog-configuration.md** | Hive, Glue, REST, Nessie, JDBC catalog setup examples |
| **hidden-partitioning-examples.md** | Partition transform examples, evolution, anti-patterns |
| **schema-evolution-details.md** | ALTER TABLE syntax, type promotion, migration patterns |
| **time-travel-advanced.md** | Rollback, cherry-pick, snapshot tagging, expiration |
| **file-formats.md** | Parquet/ORC/Avro configuration, compression comparison |
| **compaction-operations.md** | Bin-packing, sort-order, incremental compaction, scheduling |
| **write-operations-detail.md** | INSERT, MERGE, UPDATE, DELETE, transactional writes |
| **engine-integration.md** | Spark, Flink, Trino, Snowflake, Athena integration |
| **cdc-implementation.md** | Flink/Spark CDC pipelines, exactly-once semantics |
| **performance-optimization.md** | Pruning, predicate pushdown, metadata caching, sort order |
| **monitoring.md** | File health, snapshot metrics, orphan detection, alerting |
| **common-pitfalls.md** | Anti-patterns, troubleshooting, prevention checklist |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Crown Bronze layer | `s3://crown-bronze-layer/` |
| Iceberg catalog config | Glue Catalog (AWS managed) |
| CDC ingestion pipeline | Debezium → Kafka → Flink/Spark → Iceberg |
| Query engine | Starburst (Trino) |

---

## References

- [Apache Iceberg Documentation](https://iceberg.apache.org/docs/latest/)
- [Iceberg Table Spec](https://iceberg.apache.org/spec/)
- [AWS Iceberg Guide](https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/)
- [Netflix Iceberg Case Study](https://netflixtechblog.com/how-netflix-uses-iceberg-for-big-data-analytics-8c9ffc6c7e92)
- [Starburst Iceberg Integration](https://docs.starburst.io/starburst-galaxy/data-engineering/iceberg-tables.html)

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
