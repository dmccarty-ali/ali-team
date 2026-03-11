# ali-starburst - Query Patterns and Optimization

Query patterns, optimization techniques, and execution strategies.

---

## Cross-Catalog Query Patterns

### Federation Pattern

```sql
-- Join Iceberg lake with PostgreSQL operational database
SELECT
    i.event_id,
    i.event_timestamp,
    i.user_id,
    p.username,
    p.email
FROM iceberg.bronze.events i
JOIN postgresql.public.users p ON i.user_id = p.id
WHERE i.event_timestamp > CURRENT_DATE - INTERVAL '7' DAY;
```

**Execution:**
1. Coordinator splits query into tasks
2. Workers read from iceberg.bronze.events (filtered to 7 days)
3. Workers read from postgresql.public.users (full table)
4. Workers perform hash join in memory
5. Coordinator aggregates and returns

**Performance Considerations:**
- Smaller table should be PostgreSQL (broadcast join)
- Use filter on Iceberg to reduce data volume
- Consider caching PostgreSQL table if reused

### Multi-Warehouse Aggregation

```sql
-- Aggregate across Snowflake and Iceberg
WITH snowflake_sales AS (
    SELECT region, SUM(amount) AS total
    FROM snowflake.sales.fact_sales
    WHERE sale_date >= DATE '2024-01-01'
    GROUP BY region
),
iceberg_sales AS (
    SELECT region, SUM(amount) AS total
    FROM iceberg.gold.sales
    WHERE sale_date >= DATE '2024-01-01'
    GROUP BY region
)
SELECT
    COALESCE(s.region, i.region) AS region,
    COALESCE(s.total, 0) + COALESCE(i.total, 0) AS combined_total
FROM snowflake_sales s
FULL OUTER JOIN iceberg_sales i ON s.region = i.region;
```

---

## Query Optimization

### Pushdown Verification

Use EXPLAIN to verify pushdown:

```sql
EXPLAIN SELECT * FROM postgresql.public.orders
WHERE status = 'shipped' AND created_at > DATE '2024-01-01';
```

**Look for:**
- `ScanFilterProject[table = postgresql:public:orders, filterPredicate = ...]` - Filter pushed
- `RemoteQuery[query = SELECT * FROM ...]` - Full query pushed
- `TableScan[table = postgresql:public:orders]` - No pushdown (bad)

### Cost-Based Optimizer (CBO)

```sql
-- Analyze tables to gather statistics
ANALYZE iceberg.gold.sales;

-- CBO uses statistics for:
-- - Join order selection
-- - Join strategy (broadcast vs shuffle)
-- - Partition pruning
```

### Partition Pruning

```sql
-- Good: Partition filter enables pruning
SELECT * FROM iceberg.bronze.events
WHERE event_date = DATE '2024-01-15';
-- Reads only 2024-01-15 partition

-- Bad: Function on partition column prevents pruning
SELECT * FROM iceberg.bronze.events
WHERE YEAR(event_date) = 2024;
-- Reads all partitions, applies filter after

-- Fix: Rewrite to use partition column directly
SELECT * FROM iceberg.bronze.events
WHERE event_date >= DATE '2024-01-01'
  AND event_date < DATE '2025-01-01';
```

### Bucketing for Joins

```sql
-- Create bucketed table (Iceberg)
CREATE TABLE iceberg.silver.orders (
    order_id BIGINT,
    customer_id BIGINT,
    total DECIMAL(10,2)
)
WITH (
    partitioning = ARRAY['bucket(customer_id, 16)']
);

-- Bucketed join (no shuffle)
SELECT o.order_id, c.name
FROM iceberg.silver.orders o
JOIN iceberg.silver.customers c ON o.customer_id = c.customer_id;
-- If customers also bucketed on customer_id with 16 buckets, no shuffle needed
```

---

## Performance Patterns

### Broadcast Join

```sql
-- Small table broadcast to all workers
SELECT l.*, s.region_name
FROM iceberg.gold.large_fact l
JOIN iceberg.gold.small_dim s ON l.region_id = s.region_id;

-- Execution:
-- 1. Small table (small_dim) broadcast to all workers
-- 2. Large table (large_fact) read in parallel
-- 3. Each worker performs local join
```

**When to use:**
- One table < 10 MB (fits in memory)
- Other table large (GB-TB)

### Shuffle Join

```sql
-- Both tables large, shuffle required
SELECT o.*, c.*
FROM iceberg.gold.orders o
JOIN iceberg.gold.customers c ON o.customer_id = c.customer_id;

-- Execution:
-- 1. Both tables partitioned by join key (customer_id)
-- 2. Data shuffled across workers by hash(customer_id)
-- 3. Each worker joins matching partitions
```

**When to use:**
- Both tables large (> 100 MB)
- Cannot fit smaller table in memory

### Predicate Pushdown

```sql
-- Push filter to source
SELECT * FROM postgresql.public.orders
WHERE status = 'shipped'
  AND order_date >= CURRENT_DATE - INTERVAL '30' DAY;

-- Trino sends to PostgreSQL:
-- SELECT * FROM orders WHERE status = 'shipped' AND order_date >= '2024-01-01'
-- Only matching rows pulled over network
```

### Projection Pushdown

```sql
-- Select only needed columns
SELECT order_id, total FROM postgresql.public.orders;

-- Trino sends to PostgreSQL:
-- SELECT order_id, total FROM orders
-- Reduces network transfer
```

---

## EXPLAIN ANALYZE

### Detailed Execution Plan

```sql
-- Detailed execution plan with actuals
EXPLAIN ANALYZE
SELECT region, COUNT(*) FROM iceberg.gold.sales
WHERE sale_date >= DATE '2024-01-01'
GROUP BY region;
```

**Output includes:**
- Actual row counts at each stage
- CPU and memory usage per stage
- Time spent in each operator
- Connector pushdown verification

---

**Last Updated:** 2026-02-16
