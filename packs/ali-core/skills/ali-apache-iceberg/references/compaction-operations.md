# ali-apache-iceberg - Compaction Operations Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg compaction examples" or "I need detailed compaction procedures"

---

## Bin-Packing Compaction

### Basic Bin-Packing

```sql
-- Combine small files into larger files without changing sort order
CALL my_catalog.system.rewrite_data_files(
    table => 'bronze_layer.cdc_orders',
    strategy => 'binpack',
    options => map(
        'target-file-size-bytes', '536870912',  -- 512 MB target
        'min-file-size-bytes', '134217728',     -- Skip files > 128 MB
        'max-concurrent-file-group-rewrites', '4'
    )
);
```

### Partition-Specific Compaction

```python
# Compact only recent partitions
spark.sql("""
    CALL my_catalog.system.rewrite_data_files(
        table => 'bronze_layer.cdc_orders',
        where => "order_date >= current_date() - INTERVAL 7 DAYS",
        strategy => 'binpack',
        options => map('target-file-size-bytes', '536870912')
    )
""")
```

### Targeted File Size Compaction

```python
# Only compact partitions with small files
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()

# Find partitions needing compaction
partitions = spark.sql("""
    SELECT partition
    FROM my_catalog.bronze_layer.orders.files
    GROUP BY partition
    HAVING AVG(file_size_in_bytes) < 134217728  -- 128 MB
       OR COUNT(*) > 100
""").collect()

# Compact each partition
for partition in partitions:
    spark.sql(f"""
        CALL my_catalog.system.rewrite_data_files(
            'bronze_layer.orders',
            WHERE => "partition_value = '{partition.partition}'"
        )
    """)
```

### Parallel Compaction

```python
# Compact multiple partitions concurrently
from concurrent.futures import ThreadPoolExecutor

def compact_partition(partition_filter):
    spark.sql(f"""
        CALL system.rewrite_data_files(
            'orders',
            where => "{partition_filter}",
            strategy => 'binpack'
        )
    """)

partitions = [
    "order_date = '2026-01-15'",
    "order_date = '2026-01-14'",
    "order_date = '2026-01-13'"
]

with ThreadPoolExecutor(max_workers=4) as executor:
    executor.map(compact_partition, partitions)
```

---

## Sort-Order Compaction

### Set Sort Order

```sql
-- Define sort order for table
ALTER TABLE cdc_orders
WRITE ORDERED BY (customer_id, order_timestamp);
```

### Sort Compaction

```sql
-- Rewrite files with optimized sort order for query patterns
CALL my_catalog.system.rewrite_data_files(
    table => 'bronze_layer.cdc_orders',
    strategy => 'sort',
    sort_order => 'customer_id, order_timestamp',
    options => map(
        'target-file-size-bytes', '536870912',
        'partial-progress.enabled', 'true',
        'partial-progress.max-commits', '5'
    )
);
```

### Multi-Column Sort

```sql
-- Optimize for common query patterns
ALTER TABLE events
WRITE ORDERED BY (event_type, event_timestamp DESC, user_id);

-- Rewrite with sort
CALL system.rewrite_data_files(
    table => 'events',
    strategy => 'sort',
    options => map('target-file-size-bytes', '536870912')
);
```

### Z-Order Clustering (Multi-Dimensional)

```sql
-- Z-order clustering for multi-dimensional queries
CALL system.rewrite_data_files(
    table => 'events',
    strategy => 'sort',
    sort_order => 'zorder(user_id, event_type, event_timestamp)',
    options => map('target-file-size-bytes', '536870912')
);
```

---

## Incremental Compaction

### Time-Based Incremental

```python
# Compact only last 7 days
spark.sql("""
    CALL my_catalog.system.rewrite_data_files(
        table => 'bronze_layer.cdc_orders',
        where => "order_date >= current_date() - INTERVAL 7 DAYS"
    )
""")
```

### File-Count Threshold

```python
# Compact partitions with > 100 files
partitions_to_compact = spark.sql("""
    SELECT partition
    FROM bronze_layer.orders.files
    GROUP BY partition
    HAVING COUNT(*) > 100
""")

for partition in partitions_to_compact.collect():
    spark.sql(f"""
        CALL system.rewrite_data_files(
            'bronze_layer.orders',
            where => "partition = '{partition.partition}'"
        )
    """)
```

### Micro-Batch Compaction

```python
# Compact after each streaming micro-batch
def compact_partition(batch_df, batch_id):
    # Write batch
    batch_df.write.mode("append").saveAsTable("orders")

    # Compact partition
    partition_date = batch_df.select("order_date").first()[0]
    spark.sql(f"""
        CALL system.rewrite_data_files(
            'orders',
            where => "order_date = '{partition_date}'"
        )
    """)

# Streaming write with compaction
cdc_stream.writeStream \
    .foreachBatch(compact_partition) \
    .start()
```

---

## Monitor Compaction Need

### File Size Distribution

```sql
-- Check file sizes per partition
SELECT
    partition,
    COUNT(*) AS file_count,
    AVG(file_size_in_bytes) / 1024 / 1024 AS avg_file_size_mb,
    MIN(file_size_in_bytes) / 1024 / 1024 AS min_file_size_mb,
    MAX(file_size_in_bytes) / 1024 / 1024 AS max_file_size_mb
FROM my_catalog.bronze_layer.cdc_orders.files
GROUP BY partition
HAVING COUNT(*) > 100 OR AVG(file_size_in_bytes) < 134217728  -- 128 MB threshold
ORDER BY file_count DESC;
```

### Small File Detection

```sql
-- Identify partitions with small file problem
SELECT
    partition,
    COUNT(*) AS file_count,
    SUM(CASE WHEN file_size_in_bytes < 67108864 THEN 1 ELSE 0 END) AS small_files,  -- < 64 MB
    ROUND(SUM(CASE WHEN file_size_in_bytes < 67108864 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS pct_small
FROM cdc_orders.files
GROUP BY partition
HAVING pct_small > 50  -- More than 50% small files
ORDER BY small_files DESC;
```

### Compaction Benefit Estimate

```python
# Estimate storage/performance benefit
current_files = spark.sql("""
    SELECT
        COUNT(*) AS file_count,
        SUM(file_size_in_bytes) / 1024 / 1024 / 1024 AS total_gb
    FROM orders.files
""").collect()[0]

target_file_size = 512 * 1024 * 1024  # 512 MB
estimated_files_after = int(current_files.total_gb * 1024 / 512)

print(f"Current: {current_files.file_count} files")
print(f"After compaction: {estimated_files_after} files")
print(f"Reduction: {current_files.file_count - estimated_files_after} files ({(1 - estimated_files_after/current_files.file_count)*100:.1f}%)")
```

---

## Compaction Scheduling

### Hourly Bin-Packing (Production Pattern)

```bash
#!/bin/bash
# cron: 0 * * * * /path/to/compact_hourly.sh

# Compact partitions from last 2 hours
spark-submit --master yarn \
    --conf spark.sql.catalog.my_catalog=... \
    compact_job.py \
    --table bronze_layer.cdc_orders \
    --where "order_timestamp >= current_timestamp() - INTERVAL 2 HOURS"
```

### Nightly Sort Optimization

```bash
#!/bin/bash
# cron: 0 2 * * * /path/to/compact_nightly.sh

# Sort optimization for yesterday's data
spark-submit --master yarn \
    compact_job.py \
    --table bronze_layer.orders \
    --strategy sort \
    --where "order_date = current_date() - INTERVAL 1 DAY"
```

### Weekly Full Compaction

```bash
#!/bin/bash
# cron: 0 3 * * 0 /path/to/compact_weekly.sh

# Full compaction on Sunday
spark-submit --master yarn \
    --conf spark.executor.memory=8g \
    compact_job.py \
    --table bronze_layer.orders \
    --strategy binpack
```

---

## Compaction Job Template

```python
# compact_job.py
from pyspark.sql import SparkSession
import argparse

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--table', required=True)
    parser.add_argument('--where', default=None)
    parser.add_argument('--strategy', default='binpack')
    parser.add_argument('--target-file-size', default=536870912, type=int)
    args = parser.parse_args()

    spark = SparkSession.builder \
        .appName(f"Compact {args.table}") \
        .getOrCreate()

    # Build compaction call
    where_clause = f"where => '{args.where}'" if args.where else ""

    sql = f"""
        CALL system.rewrite_data_files(
            table => '{args.table}',
            {where_clause},
            strategy => '{args.strategy}',
            options => map('target-file-size-bytes', '{args.target_file_size}')
        )
    """

    print(f"Compacting {args.table}...")
    result = spark.sql(sql)
    result.show()

    print("Compaction complete")

if __name__ == '__main__':
    main()
```

---

## Compaction Best Practices

### Recommended Target File Sizes

| Data Volume | Target File Size | Rationale |
|-------------|-----------------|-----------|
| < 1 TB | 256 MB | Balance file count and overhead |
| 1-10 TB | 512 MB | Standard recommendation |
| 10-100 TB | 1 GB | Reduce metadata overhead |
| > 100 TB | 1-2 GB | Minimize file listing time |

### Compaction Frequency

| Workload | Bin-Pack Frequency | Sort Frequency |
|----------|-------------------|----------------|
| Streaming CDC | Hourly | Weekly |
| Batch ETL (nightly) | Daily | Weekly |
| Ad-hoc writes | Weekly | Monthly |

### Concurrent Rewrites

```sql
-- Set based on cluster size
ALTER TABLE orders SET TBLPROPERTIES (
    'write.max-concurrent-file-group-rewrites' = '8'  -- 8 parallel rewrites
);
```

**Recommendation**: 2-4x the number of executors, but monitor memory usage.

---

## Troubleshooting Compaction

### Issue: Compaction Takes Too Long

**Solution**: Compact incrementally

```python
# Instead of full table
CALL system.rewrite_data_files('orders')

# Compact per-partition
for partition in recent_partitions:
    CALL system.rewrite_data_files('orders', where => f"date = '{partition}'")
```

### Issue: Out of Memory During Compaction

**Solution**: Reduce target file size or concurrent rewrites

```sql
-- Reduce target file size
options => map('target-file-size-bytes', '268435456')  -- 256 MB instead of 512 MB

-- Reduce parallelism
options => map('max-concurrent-file-group-rewrites', '2')
```

### Issue: Compaction Creates Too Few Files

**Solution**: Lower target file size

```sql
-- Was: 1 GB target
options => map('target-file-size-bytes', '1073741824')

-- Now: 512 MB target
options => map('target-file-size-bytes', '536870912')
```

---

## References

- [Iceberg Compaction Documentation](https://iceberg.apache.org/docs/latest/maintenance/)
- [Spark Iceberg Procedures](https://iceberg.apache.org/docs/latest/spark-procedures/)
- [Netflix Compaction Strategy](https://netflixtechblog.com/how-netflix-uses-iceberg-for-big-data-analytics-8c9ffc6c7e92)
