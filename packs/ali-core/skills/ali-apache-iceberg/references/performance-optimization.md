# ali-apache-iceberg - Performance Optimization Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg performance optimization details" or "I need query tuning examples"

---

## Data Pruning Levels

### Three Levels of Pruning

| Level | Technique | Benefit | Query Example |
|-------|-----------|---------|---------------|
| **Partition Pruning** | Skip entire partitions via manifest list | 90%+ file reduction | `WHERE order_date = '2026-01-15'` |
| **File Pruning** | Skip files via min/max statistics | 80-95% file reduction | `WHERE customer_id = 12345` |
| **Column Pruning** | Read only needed columns from Parquet | 50-90% I/O reduction | `SELECT customer_id, total` |

### Partition Pruning Example

```sql
-- Query with partition predicate
SELECT * FROM orders
WHERE order_date = '2026-01-15'
  AND region = 'US-WEST';

-- Iceberg execution plan:
-- 1. Manifest list: 1000 manifest files → 5 files (partition filter)
-- 2. Manifest files: 10,000 data files → 50 files (partition match)
-- 3. Read only 50 data files
```

### File-Level Pruning via Statistics

```sql
-- Query uses column statistics to prune files
SELECT * FROM orders
WHERE customer_id = 12345;

-- Iceberg execution:
-- 1. Reads manifest files to get per-file min/max for customer_id
-- 2. Skips files where max(customer_id) < 12345 or min(customer_id) > 12345
-- 3. Only reads files that might contain customer_id=12345
```

### Column Pruning

```sql
-- Query only selects needed columns
SELECT customer_id, total_amount
FROM orders
WHERE order_date = '2026-01-15';

-- Parquet columnar format:
-- - Only reads customer_id and total_amount columns
-- - Skips all other columns (30+ columns not scanned)
-- - 90% I/O reduction
```

---

## Predicate Pushdown Optimization

### Multi-Level Pushdown

```sql
-- Query with multiple predicates
SELECT order_id, amount
FROM orders
WHERE order_date = '2026-01-15'  -- Partition pruning
  AND customer_id = 12345         -- File pruning
  AND amount > 100;               -- Row-level filtering

-- Iceberg optimization cascade:
-- 1. Partition pruning: Skip partitions where order_date != '2026-01-15'
-- 2. File pruning: Skip files where max(customer_id) < 12345 or min(amount) > 100
-- 3. Column pruning: Read only order_id, amount, customer_id columns
-- 4. Row filtering: Apply final predicates on surviving rows
```

### Partition Transform Optimization

```sql
-- Hidden partition transforms enable natural queries
CREATE TABLE events (
    event_id BIGINT,
    event_timestamp TIMESTAMP,
    user_id BIGINT
)
PARTITIONED BY (hours(event_timestamp), bucket(16, user_id));

-- User queries naturally
SELECT * FROM events
WHERE event_timestamp BETWEEN '2026-01-15 10:00:00' AND '2026-01-15 12:00:00'
  AND user_id = 999;

-- Iceberg computes:
-- - hour(event_timestamp) IN (2026-01-15-10, 2026-01-15-11, 2026-01-15-12)
-- - bucket(16, 999) = 7
-- Result: Prunes to 3 specific hour/bucket partitions
```

---

## Metadata Caching

### Enable Metadata Cache (Spark)

```python
# Cache manifest files and table metadata
spark.conf.set("spark.sql.iceberg.metadata-cache.enabled", "true")
spark.conf.set("spark.sql.iceberg.metadata-cache.max-entries", "1000")
spark.conf.set("spark.sql.iceberg.metadata-cache.expiration-interval-ms", "600000")  # 10 min

# Benefits:
# - Repeated queries skip metadata reads
# - Planning time reduced by 50-90%
```

### Catalog Caching (Trino)

```properties
# catalog/iceberg.properties
iceberg.metadata.cache-enabled=true
iceberg.metadata.cache-size=1000
iceberg.metadata.cache-ttl=10m
```

---

## Sort Order Optimization

### Define Sort Order for Query Patterns

```sql
-- Optimize for customer_id lookups
ALTER TABLE orders
WRITE ORDERED BY (customer_id, order_date DESC);

-- Queries filtering by customer_id benefit from sorted data
SELECT * FROM orders
WHERE customer_id = 12345
ORDER BY order_date DESC
LIMIT 10;

-- Benefits:
-- - Data locality: customer_id values clustered
-- - Early termination: LIMIT stops after 10 rows
-- - Better compression: sorted data compresses better
```

### Multi-Column Sort Order

```sql
-- Optimize for multi-dimensional queries
ALTER TABLE events
WRITE ORDERED BY (event_type, event_timestamp DESC, user_id);

-- Prioritizes:
-- 1. event_type (high selectivity)
-- 2. event_timestamp (temporal locality)
-- 3. user_id (tie-breaker)
```

### Z-Order Clustering (Multi-Dimensional)

```sql
-- Z-order for queries on multiple uncorrelated columns
ALTER TABLE user_events
WRITE ORDERED BY zorder(user_id, event_type, country_code);

-- Optimizes queries like:
-- - WHERE user_id = X
-- - WHERE event_type = Y
-- - WHERE country_code = Z
-- - WHERE user_id = X AND event_type = Y  (best case)
```

---

## File Size Optimization

### Target File Size Tuning

```sql
-- Small dataset (< 1 TB)
ALTER TABLE small_table SET TBLPROPERTIES (
    'write.target-file-size-bytes' = '268435456'  -- 256 MB
);

-- Large dataset (10-100 TB)
ALTER TABLE large_table SET TBLPROPERTIES (
    'write.target-file-size-bytes' = '1073741824'  -- 1 GB
);

-- Very large dataset (> 100 TB)
ALTER TABLE huge_table SET TBLPROPERTIES (
    'write.target-file-size-bytes' = '2147483648'  -- 2 GB
);
```

**Rationale:**
- Smaller files: Lower latency for small queries, but more metadata overhead
- Larger files: Lower metadata overhead, but slower for selective queries

---

## Parquet Optimization

### Row Group Size

```sql
-- Balance between columnar pruning and memory
ALTER TABLE orders SET TBLPROPERTIES (
    'write.parquet.row-group-size-bytes' = '134217728'  -- 128 MB (standard)
);

-- For wide tables (100+ columns):
ALTER TABLE wide_table SET TBLPROPERTIES (
    'write.parquet.row-group-size-bytes' = '268435456'  -- 256 MB
);
```

### Bloom Filters

```python
# Enable Bloom filters for high-cardinality columns
spark.sql("""
    ALTER TABLE orders SET TBLPROPERTIES (
        'write.parquet.bloom-filter-enabled.column.customer_id' = 'true',
        'write.parquet.bloom-filter-max-bytes' = '1048576'  -- 1 MB
    )
""")

# Benefits:
# - Point lookups on customer_id skip files without scanning
# - 10-100x speedup for selective queries
```

---

## Compaction Strategy for Performance

### Optimize for Read Performance

```sql
-- Bin-pack + sort for best read performance
CALL system.rewrite_data_files(
    table => 'orders',
    strategy => 'sort',
    sort_order => 'customer_id, order_date',
    options => map(
        'target-file-size-bytes', '536870912',  -- 512 MB
        'partial-progress.enabled', 'true'
    )
);
```

---

## Query Planning Optimization

### Partition Spec Evolution

```sql
-- Evolve to coarser partitioning for old data
ALTER TABLE events
REPLACE PARTITION FIELD hours(event_timestamp)
WITH days(event_timestamp)
WHERE event_timestamp < current_timestamp() - INTERVAL 90 DAYS;

-- Benefits:
-- - Fewer partitions to scan for historical queries
-- - Lower metadata overhead
-- - Faster query planning
```

### Manifest File Consolidation

```sql
-- Merge small manifest files
ALTER TABLE orders SET TBLPROPERTIES (
    'commit.manifest.min-count-to-merge' = '5',
    'commit.manifest-merge.enabled' = 'true'
);

-- Automatically merges manifest files during writes
-- Reduces manifest file count, faster planning
```

---

## Concurrent Read Optimization

### Split Planning (Spark)

```python
# Parallelize file listing across executors
spark.conf.set("spark.sql.iceberg.planning.parallelism", "8")

# Benefits:
# - Faster query planning on large tables (10,000+ files)
# - Scales planning with cluster size
```

### Vectorized Reads (Spark)

```python
# Enable vectorized Parquet reads
spark.conf.set("spark.sql.parquet.enableVectorizedReader", "true")
spark.conf.set("spark.sql.parquet.columnarReaderBatchSize", "4096")

# Benefits:
# - 2-5x faster columnar scans
# - Lower CPU usage
```

---

## Statistics Collection

### Collect Column Statistics

```sql
-- Spark automatically collects min/max/null_count per data file
-- No manual statistics collection needed

-- View statistics
SELECT
    file_path,
    record_count,
    null_value_counts,
    lower_bounds,
    upper_bounds
FROM orders.files;
```

---

## Network I/O Optimization

### S3 Optimization

```python
# Configure S3 for better performance
spark.conf.set("spark.hadoop.fs.s3a.connection.maximum", "100")
spark.conf.set("spark.hadoop.fs.s3a.threads.max", "64")
spark.conf.set("spark.hadoop.fs.s3a.fast.upload", "true")
spark.conf.set("spark.hadoop.fs.s3a.block.size", "134217728")  # 128 MB
```

---

## Query Performance Benchmarks

### Scan Performance (1 billion rows, 20 columns)

| Operation | No Optimization | With Optimization | Speedup |
|-----------|----------------|-------------------|---------|
| Full table scan | 120s | 60s (compression) | 2x |
| Partition filter | 120s | 5s (partition pruning) | 24x |
| Point lookup | 120s | 0.5s (file pruning + Bloom filter) | 240x |
| Column select (2/20) | 120s | 12s (column pruning) | 10x |
| Sorted query | 120s + sort | 10s (pre-sorted) | 12x |

**Environment:** 10-node cluster, Spark 3.5, Parquet+zstd, S3 storage

---

## Troubleshooting Slow Queries

### Issue: Slow Query Planning

**Symptom**: Query takes 30+ seconds before execution starts

**Diagnosis**:
```sql
-- Check manifest file count
SELECT COUNT(*) FROM orders.manifests;

-- Check data file count
SELECT COUNT(*) FROM orders.files;
```

**Solutions**:
- Enable metadata caching
- Increase planning parallelism
- Compact manifest files
- Expire old snapshots

---

### Issue: Slow Full Table Scan

**Symptom**: Full table scan slower than expected

**Diagnosis**:
```sql
-- Check file sizes
SELECT
    AVG(file_size_in_bytes) / 1024 / 1024 AS avg_mb,
    MIN(file_size_in_bytes) / 1024 / 1024 AS min_mb,
    MAX(file_size_in_bytes) / 1024 / 1024 AS max_mb
FROM orders.files;
```

**Solutions**:
- Compact small files (bin-packing)
- Increase target file size
- Use better compression codec (zstd)

---

### Issue: Selective Query Still Slow

**Symptom**: `WHERE customer_id = X` takes 10+ seconds

**Diagnosis**:
```sql
-- Check if statistics exist
SELECT
    file_path,
    lower_bounds,
    upper_bounds
FROM orders.files
LIMIT 5;
```

**Solutions**:
- Enable Bloom filters for customer_id
- Use sort order: `customer_id, order_date`
- Partition by bucket(customer_id)

---

## References

- [Iceberg Performance](https://iceberg.apache.org/docs/latest/performance/)
- [Spark Performance Tuning](https://spark.apache.org/docs/latest/sql-performance-tuning.html)
- [Parquet Performance](https://parquet.apache.org/docs/file-format/configurations/)
