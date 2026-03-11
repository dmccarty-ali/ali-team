# ali-apache-iceberg - Monitoring Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg monitoring queries" or "I need table health metrics"

---

## Key Metrics to Track

### Essential Monitoring Metrics

| Metric | Query | Alert Threshold | Impact |
|--------|-------|-----------------|--------|
| **File count per partition** | `SELECT COUNT(*) FROM table.files GROUP BY partition` | > 500 files | Slow query planning |
| **Average file size** | `SELECT AVG(file_size_in_bytes) FROM table.files` | < 128 MB | Poor scan performance |
| **Snapshot count** | `SELECT COUNT(*) FROM table.snapshots` | > 100 snapshots | Metadata bloat |
| **Orphan files** | Check unreferenced data files | > 1000 orphans | Wasted storage |
| **Manifest file size** | `SELECT file_size_in_bytes FROM table.manifests` | > 10 MB | Slow planning |
| **Table size** | `SELECT SUM(file_size_in_bytes) FROM table.files` | N/A | Storage costs |
| **Metadata lag** | `SELECT MAX(committed_at) FROM table.snapshots` | > 1 hour behind | CDC lag |

---

## File Health Monitoring

### File Count and Size Distribution

```sql
-- Check files needing compaction
SELECT
    partition,
    COUNT(*) AS file_count,
    SUM(file_size_in_bytes) / 1024 / 1024 / 1024 AS total_gb,
    AVG(file_size_in_bytes) / 1024 / 1024 AS avg_file_size_mb,
    MIN(file_size_in_bytes) / 1024 / 1024 AS min_file_size_mb,
    MAX(file_size_in_bytes) / 1024 / 1024 AS max_file_size_mb
FROM my_catalog.bronze_layer.orders.files
GROUP BY partition
HAVING COUNT(*) > 200 OR AVG(file_size_in_bytes) < 134217728  -- 128 MB
ORDER BY file_count DESC;
```

### Small File Detection

```sql
-- Identify partitions with small file problem
SELECT
    partition,
    COUNT(*) AS file_count,
    SUM(CASE WHEN file_size_in_bytes < 67108864 THEN 1 ELSE 0 END) AS small_files,  -- < 64 MB
    ROUND(
        SUM(CASE WHEN file_size_in_bytes < 67108864 THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS pct_small
FROM my_catalog.bronze_layer.orders.files
GROUP BY partition
HAVING pct_small > 50  -- More than 50% small files
ORDER BY small_files DESC;
```

### File Size Histogram

```sql
-- File size distribution
SELECT
    CASE
        WHEN file_size_in_bytes < 10485760 THEN '< 10 MB'
        WHEN file_size_in_bytes < 67108864 THEN '10-64 MB'
        WHEN file_size_in_bytes < 134217728 THEN '64-128 MB'
        WHEN file_size_in_bytes < 536870912 THEN '128-512 MB'
        ELSE '> 512 MB'
    END AS size_bucket,
    COUNT(*) AS file_count,
    SUM(file_size_in_bytes) / 1024 / 1024 / 1024 AS total_gb
FROM orders.files
GROUP BY 1
ORDER BY 1;
```

---

## Snapshot Monitoring

### Snapshot History

```sql
-- Review snapshot growth over time
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

### Snapshot Size Growth

```sql
-- Track data growth per snapshot
SELECT
    snapshot_id,
    committed_at,
    operation,
    total_records,
    total_data_files,
    (total_records - LAG(total_records) OVER (ORDER BY committed_at)) AS records_delta,
    (total_data_files - LAG(total_data_files) OVER (ORDER BY committed_at)) AS files_delta
FROM orders.snapshots
ORDER BY committed_at DESC
LIMIT 20;
```

### Snapshot Accumulation Alert

```python
# Check if snapshots growing too fast
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

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

## Manifest File Monitoring

### Manifest File Count

```sql
-- Monitor manifest file proliferation
SELECT
    COUNT(*) AS manifest_count,
    AVG(length) / 1024 / 1024 AS avg_size_mb,
    SUM(length) / 1024 / 1024 / 1024 AS total_gb
FROM my_catalog.bronze_layer.orders.manifests;
```

### Manifest File Size Alert

```sql
-- Large manifest files indicate need for compaction
SELECT
    path,
    length / 1024 / 1024 AS size_mb,
    added_snapshot_id
FROM orders.manifests
WHERE length > 10485760  -- > 10 MB
ORDER BY length DESC;
```

---

## Orphan File Detection

### Find Orphan Data Files

```sql
-- Orphan files: data files not referenced by any manifest
-- (Requires comparing S3 listing to Iceberg metadata)

-- Step 1: List all files in Iceberg metadata
CREATE TEMP VIEW iceberg_files AS
SELECT DISTINCT file_path
FROM my_catalog.bronze_layer.orders.files;

-- Step 2: Compare with S3 listing (external tool)
-- aws s3 ls s3://bucket/warehouse/bronze_layer/orders/data/ --recursive

-- Step 3: Orphans = (S3 files) - (Iceberg files)
```

### Orphan File Cleanup

```sql
-- Remove orphan data files (not referenced by any manifest)
CALL my_catalog.system.remove_orphan_files(
    table => 'bronze_layer.orders',
    older_than => TIMESTAMP '2026-01-08 00:00:00'  -- Safety: only remove files older than 7 days
);

-- Returns:
-- orphan_file_location         | deletion_timestamp
-- s3://bucket/.../file1.parquet | 2026-01-15 10:00:00
```

### Schedule Weekly Orphan Cleanup

```bash
#!/bin/bash
# cron: 0 2 * * 0 /path/to/cleanup_orphans.sh

spark-submit cleanup_orphans.py \
    --table bronze_layer.orders \
    --older-than "7 days"
```

---

## Metadata Size Monitoring

### Table Metadata Size

```sql
-- Check metadata.json size
SELECT
    snapshot_id,
    committed_at,
    -- Metadata file path stored in metadata log
    manifest_list
FROM orders.snapshots
ORDER BY committed_at DESC
LIMIT 1;

-- Manually check S3 object size:
-- aws s3 ls s3://bucket/warehouse/bronze_layer/orders/metadata/ --recursive --human
```

### Metadata to Data Ratio

```sql
-- Healthy ratio: metadata < 1% of data size
SELECT
    SUM(file_size_in_bytes) / 1024 / 1024 / 1024 AS data_gb,
    (SELECT COUNT(*) * 100 / 1024 / 1024 FROM orders.manifests) AS manifest_mb,
    (SELECT COUNT(*) * 50 / 1024 / 1024 FROM orders.snapshots) AS snapshot_mb
FROM orders.files;

-- If metadata > 1% of data size, compact snapshots/manifests
```

---

## Partition Health

### Partition Distribution

```sql
-- Check partition skew
SELECT
    partition,
    COUNT(*) AS file_count,
    SUM(record_count) AS total_records,
    SUM(file_size_in_bytes) / 1024 / 1024 / 1024 AS total_gb
FROM orders.files
GROUP BY partition
ORDER BY total_records DESC
LIMIT 20;
```

### Partition Cardinality

```sql
-- Detect over-partitioning (too many partitions)
SELECT COUNT(DISTINCT partition) AS partition_count
FROM orders.files;

-- Alert if > 10,000 partitions
```

---

## Query Performance Metrics

### Table Scan Statistics

```python
# Spark: Track query performance
spark.sql.execution.arrow.pyspark.enabled = True

df = spark.sql("SELECT * FROM orders WHERE order_date = '2026-01-15'")

# Trigger execution and capture metrics
df.count()

# View metrics
spark.sparkContext.statusTracker().getExecutorInfos()
```

### Iceberg Scan Metrics (Spark)

```python
# Enable Iceberg metrics
spark.conf.set("spark.sql.iceberg.vectorization.enabled", "true")
spark.conf.set("spark.sql.iceberg.planning.metrics.enabled", "true")

df = spark.sql("SELECT * FROM orders WHERE order_date = '2026-01-15'")
df.explain("extended")

# Metrics include:
# - Files scanned vs files skipped
# - Partitions scanned vs partitions skipped
# - Data read (bytes)
```

---

## Storage Cost Monitoring

### Total Table Size

```sql
-- Calculate total storage used
SELECT
    SUM(file_size_in_bytes) / 1024 / 1024 / 1024 AS total_data_gb,
    COUNT(*) AS file_count,
    SUM(file_size_in_bytes) / COUNT(*) / 1024 / 1024 AS avg_file_size_mb
FROM my_catalog.bronze_layer.orders.files;
```

### Storage Growth Rate

```sql
-- Track storage growth over time
WITH snapshot_sizes AS (
    SELECT
        DATE_TRUNC('day', committed_at) AS day,
        SUM(total_records) AS records,
        SUM(total_data_files) AS files
    FROM orders.snapshots
    GROUP BY 1
)
SELECT
    day,
    records,
    files,
    records - LAG(records) OVER (ORDER BY day) AS records_growth,
    files - LAG(files) OVER (ORDER BY day) AS files_growth
FROM snapshot_sizes
ORDER BY day DESC
LIMIT 30;
```

---

## Alerting Rules

### CloudWatch/Grafana Alert Examples

```yaml
# Alert: High file count per partition
- alert: IcebergHighFileCount
  expr: iceberg_partition_file_count > 500
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Partition {{ $labels.partition }} has {{ $value }} files"

# Alert: Small average file size
- alert: IcebergSmallFiles
  expr: iceberg_avg_file_size_mb < 128
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Table {{ $labels.table }} has small files ({{ $value }} MB avg)"

# Alert: Snapshot accumulation
- alert: IcebergSnapshotAccumulation
  expr: iceberg_snapshot_count > 100
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Table {{ $labels.table }} has {{ $value }} snapshots"

# Alert: CDC lag
- alert: IcebergCDCLag
  expr: (now() - iceberg_latest_snapshot_timestamp) > 3600  # 1 hour
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "CDC lag detected: {{ $value }} seconds behind"
```

---

## Monitoring Dashboard Template

### Key Metrics Dashboard (Grafana/CloudWatch)

**Panels:**
1. **Table Size Over Time** (line chart)
   - Query: `SUM(file_size_in_bytes) / 1GB` grouped by day

2. **File Count Per Partition** (heatmap)
   - Query: `COUNT(*) FROM files GROUP BY partition`

3. **Snapshot Accumulation** (area chart)
   - Query: `COUNT(*) FROM snapshots` over time

4. **Compaction Health** (gauge)
   - Query: `% files < 128 MB`

5. **Orphan File Count** (single stat)
   - Query: Orphan file count (external script)

6. **CDC Lag** (line chart)
   - Query: `now() - MAX(cdc_timestamp)`

---

## References

- [Iceberg Metadata Tables](https://iceberg.apache.org/docs/latest/spark-queries/#metadata-tables)
- [Iceberg Maintenance](https://iceberg.apache.org/docs/latest/maintenance/)
- [Prometheus Iceberg Exporter](https://github.com/tabular-io/iceberg-prometheus-exporter)
