# ali-apache-iceberg - Write Operations Detail Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg write operation examples" or "I need detailed INSERT/MERGE syntax"

---

## Append (Insert Only)

### Basic Append

```sql
-- Spark SQL
INSERT INTO my_catalog.bronze_layer.events
SELECT * FROM staging_events;
```

### Append with DataFrame API

```python
# Spark DataFrame API
df.writeTo("my_catalog.bronze_layer.events") \
    .append()
```

### Streaming Append

```python
# Spark Structured Streaming
stream_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", "s3://checkpoints/events") \
    .toTable("my_catalog.bronze_layer.events")
```

### Append with Partitioning

```python
# Append to specific partitions
df.writeTo("events") \
    .partitionedBy("event_date", "event_type") \
    .append()
```

---

## Overwrite Operations

### Overwrite Entire Table

```sql
-- Overwrite all data
INSERT OVERWRITE TABLE events
SELECT * FROM staging_events;
```

### Overwrite Specific Partition

```sql
-- Overwrite specific partition (static partition)
INSERT OVERWRITE TABLE events
PARTITION (event_date = '2026-01-15')
SELECT * FROM staging_events
WHERE event_date = '2026-01-15';
```

### Dynamic Partition Overwrite

```python
# Overwrite only partitions present in DataFrame
df.writeTo("events") \
    .overwritePartitions()
```

### Overwrite with DataFrame API

```python
# Replace all data
df.writeTo("events") \
    .overwrite()

# Replace specific partitions
df.writeTo("events") \
    .overwritePartitions()
```

---

## Upsert (MERGE INTO)

### Basic MERGE INTO

```sql
MERGE INTO my_catalog.bronze_layer.customers target
USING staging_customers source
ON target.customer_id = source.customer_id
WHEN MATCHED AND source.updated_at > target.updated_at THEN
    UPDATE SET
        target.customer_name = source.customer_name,
        target.email = source.email,
        target.updated_at = source.updated_at
WHEN NOT MATCHED THEN
    INSERT (customer_id, customer_name, email, updated_at)
    VALUES (source.customer_id, source.customer_name, source.email, source.updated_at);
```

### MERGE with DELETE

```sql
-- Update, insert, and delete based on conditions
MERGE INTO orders target
USING order_updates source
ON target.order_id = source.order_id
WHEN MATCHED AND source.status = 'CANCELLED' THEN
    DELETE
WHEN MATCHED THEN
    UPDATE SET
        target.status = source.status,
        target.updated_at = source.updated_at
WHEN NOT MATCHED THEN
    INSERT (order_id, status, updated_at)
    VALUES (source.order_id, source.status, source.updated_at);
```

### Conditional MERGE

```sql
-- Only merge if conditions met
MERGE INTO inventory target
USING inventory_updates source
ON target.product_id = source.product_id
WHEN MATCHED AND source.quantity > target.quantity THEN
    UPDATE SET target.quantity = source.quantity  -- Only update if increasing
WHEN NOT MATCHED THEN
    INSERT *;
```

### MERGE with Complex Join

```sql
-- Multi-column join condition
MERGE INTO user_preferences target
USING preference_updates source
ON target.user_id = source.user_id
   AND target.preference_key = source.preference_key
WHEN MATCHED THEN
    UPDATE SET target.preference_value = source.preference_value
WHEN NOT MATCHED THEN
    INSERT *;
```

---

## UPDATE Operations

### Basic UPDATE

```sql
-- Update rows matching condition
UPDATE orders
SET status = 'SHIPPED', shipped_at = current_timestamp()
WHERE status = 'PENDING' AND created_at < current_date() - INTERVAL 1 DAY;
```

### UPDATE with Subquery

```sql
-- Update based on subquery
UPDATE orders
SET discount_applied = true
WHERE customer_id IN (
    SELECT customer_id FROM premium_customers
);
```

---

## DELETE Operations

### Basic DELETE

```sql
-- Delete rows matching condition
DELETE FROM events
WHERE event_timestamp < current_timestamp() - INTERVAL 90 DAYS;
```

### DELETE with Subquery

```sql
-- Delete based on subquery
DELETE FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM deleted_customers
);
```

### Partition DELETE

```python
# Delete entire partition efficiently
spark.sql("""
    DELETE FROM events
    WHERE event_date = '2025-01-01'
""")
```

---

## Copy-on-Write vs Merge-on-Read

### Copy-on-Write Configuration (Default)

```sql
-- Explicitly set copy-on-write (default behavior)
CREATE TABLE orders (
    order_id BIGINT,
    status STRING,
    updated_at TIMESTAMP
)
USING iceberg
TBLPROPERTIES (
    'write.update.mode' = 'copy-on-write',
    'write.delete.mode' = 'copy-on-write',
    'write.merge.mode' = 'copy-on-write'
);
```

**Characteristics:**
- Rewrites entire data file on UPDATE/DELETE
- High write cost
- Low read cost (no merging needed)
- Best for batch workloads

### Merge-on-Read Configuration

```sql
-- Configure for streaming CDC
CREATE TABLE cdc_orders (
    order_id BIGINT,
    status STRING,
    updated_at TIMESTAMP
)
USING iceberg
TBLPROPERTIES (
    'write.update.mode' = 'merge-on-read',
    'write.delete.mode' = 'merge-on-read',
    'write.merge.mode' = 'merge-on-read',
    'format-version' = '2'  -- Required for merge-on-read
);
```

**Characteristics:**
- Appends delete/update metadata (positional deletes)
- Low write cost
- Medium read cost (merge at read time)
- Best for streaming CDC workloads

### Performance Comparison

| Workload | Copy-on-Write | Merge-on-Read |
|----------|--------------|---------------|
| Batch UPDATE (1% rows) | Rewrite entire file | Append delete file |
| Streaming UPSERT | High latency | Low latency |
| Read performance | Fast (no merge) | Slower (merge needed) |
| Storage overhead | None | Delete files accumulate |

---

## Transactional Writes

### Atomic Multi-Table Writes (Nessie)

```python
# Nessie provides multi-table ACID transactions
spark.sql("USE BRANCH transaction_1 IN nessie")

# Write to multiple tables atomically
spark.sql("INSERT INTO orders SELECT * FROM staging_orders")
spark.sql("INSERT INTO order_items SELECT * FROM staging_items")
spark.sql("UPDATE inventory SET quantity = quantity - sold_quantity")

# Commit all or rollback all
spark.sql("MERGE BRANCH transaction_1 INTO main IN nessie")
```

### Conditional Write (Optimistic Concurrency)

```python
# Prevent concurrent write conflicts
try:
    df.writeTo("orders") \
        .option("write.wap.enabled", "true") \
        .option("write.wap.id", "batch-123") \
        .append()
except Exception as e:
    print(f"Write conflict: {e}")
```

---

## Write Performance Tuning

### Target File Size

```sql
-- Configure target file size for writes
ALTER TABLE orders SET TBLPROPERTIES (
    'write.target-file-size-bytes' = '536870912'  -- 512 MB
);
```

### Fanout Writers (High Cardinality Partitions)

```python
# Enable fanout for high-cardinality partitions
df.writeTo("events") \
    .option("fanout-enabled", "true") \
    .append()
```

**When to use:**
- Writing to > 100 partitions simultaneously
- High-cardinality partition columns (e.g., hour partitioning)

### Write Distribution Mode

```python
# Control write parallelism
spark.conf.set("spark.sql.iceberg.write.distribution-mode", "hash")

# Options:
# - none: No repartitioning (use existing parallelism)
# - hash: Hash partition by partition columns
# - range: Range partition (for sorted writes)
```

---

## Incremental Write Patterns

### Incremental MERGE (Delta Detection)

```python
# Only merge changed rows
changed_rows = spark.sql("""
    SELECT s.*
    FROM staging_customers s
    LEFT JOIN customers c ON s.customer_id = c.customer_id
    WHERE c.customer_id IS NULL  -- New customer
       OR s.updated_at > c.updated_at  -- Updated customer
""")

changed_rows.createOrReplaceTempView("changes")

spark.sql("""
    MERGE INTO customers target
    USING changes source
    ON target.customer_id = source.customer_id
    WHEN MATCHED THEN UPDATE SET *
    WHEN NOT MATCHED THEN INSERT *
""")
```

### Incremental DELETE (Expire Old Data)

```python
# Delete data older than retention period
spark.sql("""
    DELETE FROM events
    WHERE event_timestamp < current_timestamp() - INTERVAL 90 DAYS
""")
```

---

## Write Validation

### Pre-Write Validation

```python
# Validate before writing
def validate_and_write(df, table):
    # Check for duplicates
    duplicate_count = df.groupBy("customer_id").count().filter("count > 1").count()
    if duplicate_count > 0:
        raise ValueError(f"Found {duplicate_count} duplicate customer_id values")

    # Check for nulls in required fields
    null_count = df.filter("customer_id IS NULL OR email IS NULL").count()
    if null_count > 0:
        raise ValueError(f"Found {null_count} rows with null required fields")

    # Write
    df.writeTo(table).append()
```

### Post-Write Validation

```python
# Verify write succeeded
before_count = spark.table("customers").count()
df.writeTo("customers").append()
after_count = spark.table("customers").count()

expected_count = before_count + df.count()
if after_count != expected_count:
    raise ValueError(f"Write verification failed: expected {expected_count}, got {after_count}")
```

---

## Write Patterns Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| **Small appends** | Creates too many small files | Batch writes, enable auto-compaction |
| **Full table overwrite** | Deletes all snapshots, expensive | Use partition overwrite or MERGE |
| **No partition pruning** | Writes to all partitions | Use partitionedBy() with filters |
| **Copy-on-write for CDC** | High write latency | Use merge-on-read for streaming |
| **No write validation** | Bad data written | Validate before write |
| **Ignoring conflicts** | Lost updates | Use optimistic concurrency control |

---

## References

- [Iceberg Writes Documentation](https://iceberg.apache.org/docs/latest/spark-writes/)
- [MERGE INTO Syntax](https://iceberg.apache.org/docs/latest/spark-writes/#merge-into)
- [Write Performance Tuning](https://iceberg.apache.org/docs/latest/performance/)
