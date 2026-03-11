# ali-apache-iceberg - Common Pitfalls Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me common Iceberg pitfalls" or "What are Iceberg anti-patterns"

---

## Common Pitfalls

| Pitfall | Problem | Impact | Solution |
|---------|---------|--------|----------|
| **No compaction** | Thousands of small files accumulate | Slow query planning (minutes), poor scan performance, high S3 costs | Schedule hourly bin-packing compaction |
| **Snapshot explosion** | Hundreds of snapshots, metadata bloat | Metadata reads slow, table history unmanageable | Configure snapshot expiration (7-30 days retention) |
| **Wrong partition granularity** | Too coarse (no pruning) or too fine (too many files) | Ineffective partition pruning or excessive metadata | Use day/hour partitioning, monitor file count per partition |
| **Ignoring sort order** | Random data layout, poor locality | Slower queries, poor compression | Set WRITE ORDERED BY for common query patterns |
| **Using copy-on-write for CDC** | High write latency, file churn | Streaming lag, increased costs | Use merge-on-read for streaming CDC ingestion |
| **Not monitoring file sizes** | Performance degrades over time unnoticed | Gradually slower queries | Alert on avg file size < 128 MB |
| **Forgetting orphan cleanup** | Storage costs grow unbounded | Wasted storage, higher S3 costs | Schedule weekly orphan file removal |
| **No partition evolution** | Stuck with initial partition scheme forever | Poor performance as data volume grows | Use ALTER TABLE to evolve partitioning |
| **Exposing partition columns** | Users write Hive-style queries with partition filters | Lose hidden partitioning benefit | Use hidden partitioning, educate users |
| **Hardcoded metadata location** | Portability issues, environment coupling | Can't promote dev → staging → prod | Use catalog abstractions (Glue, REST) |
| **No schema evolution strategy** | Breaking schema changes, data pipeline failures | Downtime, emergency migrations | Plan schema evolution upfront, use safe widening |
| **Copy-on-write with high update volume** | Entire file rewritten for single row update | Very high write costs, slow CDC | Use merge-on-read mode |
| **Too many concurrent writers** | Snapshot conflicts, write failures | Failed jobs, retries | Limit concurrent writes, use partition fanout |

---

## Detailed Pitfall Analysis

### 1. No Compaction Schedule

**Symptoms:**
- Query planning takes 30+ seconds
- Queries slower over time
- High S3 LIST operation costs

**Root Cause:**
- Streaming writes create thousands of small files
- No automated compaction job

**Example:**
```sql
-- After 1 month of streaming CDC:
SELECT COUNT(*) FROM orders.files;
-- Result: 50,000 files (500 MB total, 10 KB per file)

-- Query planning:
-- - Lists 50,000 S3 objects
-- - Reads 50,000 manifest entries
-- - Takes 45 seconds before query executes
```

**Solution:**
```bash
# Schedule hourly compaction
cron: 0 * * * * /usr/local/bin/compact-iceberg.sh

# compact-iceberg.sh
spark-submit \
    --conf spark.sql.catalog.my_catalog=... \
    compact_job.py \
    --table bronze_layer.orders \
    --where "order_date >= current_date() - INTERVAL 2 DAYS"
```

---

### 2. Snapshot Explosion

**Symptoms:**
- Table metadata.json files growing > 10 MB
- Snapshot history query slow
- S3 listing shows thousands of metadata files

**Root Cause:**
- High-frequency writes (every minute)
- No snapshot expiration configured
- Snapshots retained indefinitely

**Example:**
```sql
-- After 1 month:
SELECT COUNT(*) FROM orders.snapshots;
-- Result: 43,200 snapshots (1 per minute)

-- Metadata overhead:
-- - 43,200 manifest list files
-- - 43,200 entries in metadata.json snapshot history
-- - 500 MB of metadata (vs 100 GB of data)
```

**Solution:**
```sql
-- Configure automatic expiration
ALTER TABLE orders SET TBLPROPERTIES (
    'history.expire.max-snapshot-age-ms' = '604800000',  -- 7 days
    'history.expire.min-snapshots-to-keep' = '10'
);

-- Manual cleanup
CALL system.expire_snapshots('orders', TIMESTAMP '2026-01-08');
```

---

### 3. Wrong Partition Granularity

**Too Coarse (No Pruning):**
```sql
-- BAD: One partition per year for daily queries
CREATE TABLE events (ts TIMESTAMP, data STRING)
PARTITIONED BY (years(ts));

-- Query:
SELECT * FROM events WHERE ts = '2026-01-15';

-- Problem: Scans entire year partition (365 days of data)
```

**Too Fine (Too Many Partitions):**
```sql
-- BAD: Minute-level partitioning for low-frequency data
CREATE TABLE daily_reports (ts TIMESTAMP, data STRING)
PARTITIONED BY (minutes(ts));  -- 1,440 partitions per day

-- Problem: 525,600 partitions per year, excessive metadata
```

**Solution:**
```sql
-- GOOD: Match partition granularity to query patterns
CREATE TABLE events (ts TIMESTAMP, data STRING)
PARTITIONED BY (days(ts));  -- Daily queries = daily partitions
```

---

### 4. Ignoring Sort Order

**Without Sort Order:**
```sql
CREATE TABLE orders (customer_id BIGINT, order_date DATE, amount DECIMAL);

-- Data is randomly distributed:
-- File 1: customer_id 1, 500, 999, 1234, ...
-- File 2: customer_id 3, 7, 1001, 5000, ...

-- Query:
SELECT * FROM orders WHERE customer_id = 12345 LIMIT 10;

-- Problem: Must scan ALL files to find customer_id = 12345
```

**With Sort Order:**
```sql
ALTER TABLE orders WRITE ORDERED BY (customer_id, order_date);

-- After compaction:
-- File 1: customer_id 1-100
-- File 2: customer_id 101-200
-- File 3: customer_id 201-300
-- ...

-- Query:
SELECT * FROM orders WHERE customer_id = 12345 LIMIT 10;

-- Benefit: File pruning skips 99% of files, reads only file containing 12345
```

---

### 5. Copy-on-Write for CDC (High Update Volume)

**Problem:**
```sql
-- Copy-on-write mode (default)
CREATE TABLE users (id BIGINT, name STRING);

-- CDC update (1 row changed out of 100,000 in file):
UPDATE users SET name = 'New Name' WHERE id = 12345;

-- What happens:
-- 1. Read entire 512 MB data file
-- 2. Modify 1 row
-- 3. Rewrite entire 512 MB file
-- Result: 512 MB write for 1-row change
```

**Solution:**
```sql
-- Use merge-on-read for CDC
CREATE TABLE users (id BIGINT, name STRING)
TBLPROPERTIES (
    'write.update.mode' = 'merge-on-read',
    'write.delete.mode' = 'merge-on-read',
    'format-version' = '2'
);

-- CDC update:
-- 1. Append delete marker (100 KB)
-- 2. Append new row (100 KB)
-- Result: 200 KB write for 1-row change (2,560x less)
```

---

### 6. Not Monitoring File Sizes

**Scenario:**
- Initial writes create 512 MB files
- 6 months later: average file size drops to 10 MB
- Nobody notices until queries slow down 50x

**Prevention:**
```python
# Alert on small file accumulation
avg_file_size = spark.sql("""
    SELECT AVG(file_size_in_bytes) / 1024 / 1024 AS avg_mb
    FROM orders.files
""").collect()[0][0]

if avg_file_size < 128:
    alert(f"WARNING: Avg file size {avg_file_size} MB (should be 512 MB)")
```

---

### 7. Forgetting Orphan Cleanup

**Problem:**
- Compaction creates new files, deletes old file references
- Old files remain on S3 (orphaned)
- Storage grows indefinitely

**Example:**
```bash
# After 1 year:
aws s3 ls s3://bucket/warehouse/orders/data/ --recursive | wc -l
# Result: 500,000 files

# But Iceberg only references:
SELECT COUNT(*) FROM orders.files;
# Result: 10,000 files

# Orphans: 490,000 files (98% waste)
```

**Solution:**
```bash
# Weekly cleanup
cron: 0 2 * * 0 cleanup-orphans.sh

# cleanup-orphans.sh
spark-sql -e "CALL system.remove_orphan_files('orders', TIMESTAMP '$(date -d '7 days ago' +%Y-%m-%d)');"
```

---

### 8. No Partition Evolution Strategy

**Problem:**
- Start with day partitions
- Data grows 100x
- Now have 10,000 partitions (excessive metadata)
- Can't change partitioning without rewrite

**Example:**
```sql
-- Initial (100 GB data):
CREATE TABLE events (ts TIMESTAMP, data STRING)
PARTITIONED BY (days(ts));  -- 365 partitions/year

-- After 10 years (10 TB data):
SELECT COUNT(DISTINCT partition) FROM events.files;
-- Result: 3,650 partitions (excessive metadata)
```

**Solution:**
```sql
-- Evolve to month partitions for old data
ALTER TABLE events
REPLACE PARTITION FIELD days(ts)
WITH months(ts);

-- New data: month partitions
-- Old data: day partitions (no rewrite needed)
-- Result: 120 new partitions/year instead of 365
```

---

### 9. Exposing Partition Columns (Hive-Style Queries)

**Anti-Pattern:**
```sql
-- User writes Hive-style query
SELECT * FROM events
WHERE year = 2026 AND month = 1 AND day = 15;
```

**Why It's Bad:**
- Users must know partition scheme
- Breaks abstraction (hidden partitioning benefit lost)
- Query changes required if partitioning evolves

**Correct Pattern:**
```sql
-- User queries naturally (Iceberg computes partitions)
SELECT * FROM events
WHERE event_timestamp = '2026-01-15';

-- Iceberg automatically:
-- 1. Computes year=2026, month=1, day=15 from event_timestamp
-- 2. Prunes to correct partition
```

---

### 10. Too Many Concurrent Writers

**Problem:**
```python
# 100 Spark jobs writing to same table simultaneously
for i in range(100):
    spark_job(f"INSERT INTO orders SELECT * FROM source_{i}")

# Result: Snapshot conflicts, write failures
# ERROR: Concurrent modification detected, retry...
```

**Solution:**
```python
# Limit concurrent writes
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=4) as executor:
    executor.map(write_job, range(100))

# Or use partition fanout for high-cardinality writes
df.writeTo("orders").option("fanout-enabled", "true").append()
```

---

## Prevention Checklist

- [ ] Compaction scheduled (hourly bin-pack, weekly sort)
- [ ] Snapshot expiration configured (7-30 days)
- [ ] Partition granularity matches query patterns
- [ ] Sort order set for common queries
- [ ] Merge-on-read enabled for CDC tables
- [ ] File size monitoring (alert < 128 MB avg)
- [ ] Orphan cleanup scheduled (weekly)
- [ ] Partition evolution strategy planned
- [ ] Users educated on hidden partitioning (no year/month/day filters)
- [ ] Concurrent write limits enforced

---

## References

- [Iceberg Maintenance](https://iceberg.apache.org/docs/latest/maintenance/)
- [Iceberg Performance](https://iceberg.apache.org/docs/latest/performance/)
- [Netflix Iceberg Lessons Learned](https://netflixtechblog.com/how-netflix-uses-iceberg-for-big-data-analytics-8c9ffc6c7e92)
