# ali-snowflake-data-engineer - Storage Analysis Reference

## Table Storage Metrics

```sql
-- Table storage metrics
SELECT
    table_name,
    active_bytes/1e9 as active_gb,
    time_travel_bytes/1e9 as time_travel_gb,
    failsafe_bytes/1e9 as failsafe_gb,
    retained_for_clone_bytes/1e9 as clone_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE table_catalog = 'MY_DB'
  AND table_schema = 'MY_SCHEMA'
ORDER BY active_bytes DESC;

-- Partition statistics
SELECT
    TABLE_NAME,
    ROW_COUNT,
    BYTES,
    BYTES/NULLIF(ROW_COUNT, 0) as avg_row_bytes
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'MY_SCHEMA';
```

## Advanced Cloning

```sql
-- Clone with Time Travel
CREATE TABLE restored_table CLONE production_table
  AT (TIMESTAMP => '2024-01-15 00:00:00'::TIMESTAMP);

-- Clone schema (includes all objects)
CREATE SCHEMA dev_schema CLONE prod_schema;

-- Clone database for testing
CREATE DATABASE test_db CLONE prod_db;

-- Track clone lineage
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE referenced_object_name = 'production_table'
  AND dependency_type = 'BY_ID';
```
