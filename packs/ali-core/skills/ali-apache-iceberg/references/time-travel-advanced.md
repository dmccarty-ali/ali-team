# ali-apache-iceberg - Time Travel Advanced Operations Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg rollback examples" or "I need snapshot expiration configuration"

---

## Rollback Operations

### Rollback to Previous Snapshot

```sql
-- Rollback to specific snapshot ID
CALL my_catalog.system.rollback_to_snapshot('bronze_layer.orders', 1234567889);

-- Result:
-- - Current snapshot becomes 1234567889
-- - Newer snapshots still exist (not deleted)
-- - Can roll forward again if needed
```

### Rollback to Timestamp

```sql
-- Rollback to state at specific time
CALL my_catalog.system.rollback_to_timestamp(
    'bronze_layer.orders',
    TIMESTAMP '2026-01-14 18:00:00'
);

-- Finds snapshot closest to timestamp and rolls back
```

### Rollback with Validation

```python
# Python: Validate before rollback
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

# Check snapshot history
snapshots = spark.sql("""
    SELECT snapshot_id, committed_at, operation
    FROM my_catalog.bronze_layer.orders.snapshots
    ORDER BY committed_at DESC
""").collect()

# Find target snapshot
target_snapshot = None
for snapshot in snapshots:
    if snapshot.operation == 'append' and snapshot.committed_at < target_time:
        target_snapshot = snapshot.snapshot_id
        break

# Rollback
if target_snapshot:
    spark.sql(f"""
        CALL my_catalog.system.rollback_to_snapshot(
            'bronze_layer.orders',
            {target_snapshot}
        )
    """)
```

---

## Cherry-Pick Operations

### Cherry-Pick Specific Snapshot

```sql
-- Reapply changes from a specific snapshot
CALL my_catalog.system.cherrypick_snapshot('bronze_layer.orders', 1234567890);

-- Use case: Accidentally reverted a commit, want to re-apply it
```

### Cherry-Pick Workflow Example

```sql
-- Scenario: Rollback went too far, need to recover specific change

-- Step 1: Rollback to safe point
CALL system.rollback_to_snapshot('orders', 100);

-- Step 2: Cherry-pick the one good change
CALL system.cherrypick_snapshot('orders', 105);

-- Result: State at snapshot 100 + changes from snapshot 105
```

---

## Snapshot Expiration

### Basic Expiration

```sql
-- Expire snapshots older than 7 days
CALL my_catalog.system.expire_snapshots(
    'bronze_layer.orders',
    TIMESTAMP '2026-01-08 00:00:00'
);

-- Deletes:
-- - Snapshot metadata
-- - Manifest files no longer referenced
-- - Data files no longer referenced
```

### Expiration with Retention

```sql
-- Expire old snapshots but keep minimum
ALTER TABLE orders SET TBLPROPERTIES (
    'history.expire.max-snapshot-age-ms' = '604800000',  -- 7 days in ms
    'history.expire.min-snapshots-to-keep' = '10'
);

-- Keeps at least 10 snapshots even if older than 7 days
```

### Configure Snapshot Retention

```sql
-- Comprehensive retention policy
ALTER TABLE orders SET TBLPROPERTIES (
    'history.expire.min-snapshots-to-keep' = '10',
    'history.expire.max-snapshot-age-ms' = '2592000000',  -- 30 days
    'history.expire.max-ref-age-ms' = '7776000000'  -- 90 days for tagged snapshots
);
```

### Automated Expiration

```python
# Schedule daily snapshot expiration (PySpark)
from datetime import datetime, timedelta

cutoff = datetime.now() - timedelta(days=7)
cutoff_ts = cutoff.strftime('%Y-%m-%d %H:%M:%S')

spark.sql(f"""
    CALL my_catalog.system.expire_snapshots(
        'bronze_layer.orders',
        TIMESTAMP '{cutoff_ts}',
        older_than => TIMESTAMP '{cutoff_ts}',
        retain_last => 10
    )
""")
```

---

## Snapshot Tagging

### Create Named Snapshot Tag

```sql
-- Tag current snapshot for long-term retention
ALTER TABLE orders
CREATE TAG backup_2026_01_15 AS OF SNAPSHOT 1234567890
RETAIN 365 DAYS;

-- Tag persists beyond normal expiration
```

### Query Tagged Snapshot

```sql
-- Query by tag name
SELECT * FROM orders
FOR SYSTEM_VERSION AS OF 'backup_2026_01_15';
```

### List Tags

```sql
-- Show all tags
SELECT * FROM my_catalog.bronze_layer.orders.refs
WHERE type = 'TAG';

-- Output:
-- name                | snapshot_id | type | max_ref_age_ms
-- backup_2026_01_15  | 1234567890  | TAG  | 31536000000  (365 days)
```

### Drop Tag

```sql
-- Remove tag (allows snapshot to expire normally)
ALTER TABLE orders DROP TAG backup_2026_01_15;
```

---

## Snapshot Branching (Nessie)

### Create Branch

```sql
-- Create development branch from main
CREATE BRANCH dev_branch FROM main IN nessie_catalog;

-- Switch to branch
USE BRANCH dev_branch IN nessie_catalog;
```

### Merge Branch

```sql
-- Merge development changes back to main
MERGE BRANCH dev_branch INTO main IN nessie_catalog;
```

### Branch for Testing

```python
# Test changes in isolated branch
spark.sql("CREATE BRANCH test_schema_change FROM main IN nessie")
spark.sql("USE BRANCH test_schema_change IN nessie")

# Make schema change
spark.sql("ALTER TABLE orders ADD COLUMN new_field STRING")

# Test queries
df = spark.sql("SELECT * FROM orders WHERE new_field IS NOT NULL")

# If successful, merge to main
spark.sql("MERGE BRANCH test_schema_change INTO main IN nessie")

# If failed, drop branch
spark.sql("DROP BRANCH test_schema_change IN nessie")
```

---

## Data File Retention

### Configure Data File Retention

```sql
-- Keep data files for 30 days even if unreferenced
ALTER TABLE orders SET TBLPROPERTIES (
    'write.data.retention-period-ms' = '2592000000'  -- 30 days
);

-- Allows time travel up to 30 days back
```

### Check Data File References

```sql
-- Find data files not in current snapshot
SELECT file_path
FROM my_catalog.bronze_layer.orders.all_data_files
WHERE file_path NOT IN (
    SELECT file_path
    FROM my_catalog.bronze_layer.orders.files
);

-- These files are candidates for expiration
```

---

## Metadata Expiration

### Expire Old Metadata

```sql
-- Clean up old metadata files
CALL my_catalog.system.expire_snapshots(
    'bronze_layer.orders',
    older_than => TIMESTAMP '2026-01-01',
    retain_last => 5,
    max_concurrent_deletes => 10
);

-- Removes:
-- - metadata.json files
-- - manifest list files
-- - manifest files
-- - unreferenced data files
```

---

## Monitoring Snapshot Growth

### Snapshot Statistics

```sql
-- Analyze snapshot accumulation
SELECT
    DATE_TRUNC('day', committed_at) AS day,
    COUNT(*) AS snapshot_count,
    SUM(total_data_files) AS total_files,
    SUM(total_records) AS total_records,
    MAX(total_data_files) - MIN(total_data_files) AS file_churn
FROM my_catalog.bronze_layer.orders.snapshots
WHERE committed_at > current_timestamp() - INTERVAL 30 DAYS
GROUP BY 1
ORDER BY 1 DESC;
```

### Alert on Snapshot Explosion

```python
# Check if snapshots growing too fast
snapshot_count = spark.sql("""
    SELECT COUNT(*)
    FROM my_catalog.bronze_layer.orders.snapshots
    WHERE committed_at > current_timestamp() - INTERVAL 1 DAY
""").collect()[0][0]

if snapshot_count > 1000:
    print(f"WARNING: {snapshot_count} snapshots created in last 24 hours")
    print("Consider increasing snapshot expiration frequency")
```

---

## Rollback Safety Checklist

Before rollback, verify:
1. **Downstream dependencies**: Will rollback break downstream jobs reading current data?
2. **Snapshot inspection**: Review snapshot history to confirm target snapshot
3. **Backup tag**: Create tag on current snapshot for recovery
4. **Test in branch**: Use Nessie branch to test rollback before main
5. **Concurrent writers**: Pause writes during rollback to avoid conflicts

---

## References

- [Iceberg Snapshots](https://iceberg.apache.org/docs/latest/branching/)
- [Snapshot Expiration](https://iceberg.apache.org/docs/latest/maintenance/#expire-snapshots)
- [Nessie Branching](https://projectnessie.org/docs/)
