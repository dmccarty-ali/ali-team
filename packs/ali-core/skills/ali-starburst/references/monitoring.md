# ali-starburst - Monitoring and Performance Analysis

Monitoring queries, metrics, and performance analysis techniques.

---

## Key Metrics

| Metric | What It Means | Target |
|--------|--------------|--------|
| **Query latency (P95)** | 95th percentile query time | < 10s (interactive), < 60s (batch) |
| **Queue time** | Time waiting for resources | < 1s |
| **CPU utilization** | Worker CPU usage | 60-80% (allows headroom) |
| **Memory pressure** | Out-of-memory kills | 0 per day |
| **Connector latency** | Time to fetch from source | Varies by connector |
| **Cache hit rate** | Warp Speed effectiveness | > 50% for hot queries |

---

## Query History Analysis

### Recent Query Performance

```sql
-- Query performance analysis
SELECT
    query_id,
    query,
    state,
    queued_time_ms,
    analysis_time_ms,
    execution_time_ms,
    total_cpu_time_ms,
    total_memory_bytes,
    user,
    source
FROM system.runtime.queries
WHERE created > CURRENT_TIMESTAMP - INTERVAL '1' HOUR
ORDER BY execution_time_ms DESC
LIMIT 20;
```

### Failed Queries

```sql
-- Find failed queries
SELECT
    query_id,
    query,
    error_type,
    error_code,
    failure_message,
    user,
    created
FROM system.runtime.queries
WHERE state = 'FAILED'
  AND created > CURRENT_TIMESTAMP - INTERVAL '24' HOUR
ORDER BY created DESC;
```

---

## Slow Query Analysis

### Queries with Poor Pushdown

```sql
-- Find queries with lots of data read (indicates poor pushdown)
SELECT
    query_id,
    query,
    physical_input_bytes / 1024 / 1024 / 1024 AS input_gb,
    output_rows,
    execution_time_ms / 1000 AS execution_sec
FROM system.runtime.queries
WHERE physical_input_bytes > 10 * 1024 * 1024 * 1024  -- > 10 GB
ORDER BY physical_input_bytes DESC;
```

### Memory-Intensive Queries

```sql
-- Find queries consuming excessive memory
SELECT
    query_id,
    query,
    total_memory_bytes / 1024 / 1024 / 1024 AS memory_gb,
    execution_time_ms / 1000 AS execution_sec,
    state
FROM system.runtime.queries
WHERE total_memory_bytes > 50 * 1024 * 1024 * 1024  -- > 50 GB
ORDER BY total_memory_bytes DESC;
```

### Long-Running Queries

```sql
-- Find currently running queries over 5 minutes
SELECT
    query_id,
    query,
    state,
    execution_time_ms / 1000 / 60 AS execution_minutes,
    user,
    source
FROM system.runtime.queries
WHERE state = 'RUNNING'
  AND execution_time_ms > 300000  -- > 5 minutes
ORDER BY execution_time_ms DESC;
```

---

## Connector Performance

### Connector Read Latency

```sql
-- Analyze connector read performance
SELECT
    split_source AS connector,
    COUNT(*) AS read_count,
    AVG(physical_input_bytes) / 1024 / 1024 AS avg_mb_read,
    AVG(physical_input_read_time_ms) AS avg_read_time_ms
FROM system.runtime.tasks
WHERE created > CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY split_source
ORDER BY avg_read_time_ms DESC;
```

### Pushdown Effectiveness

```sql
-- Compare input vs output bytes (pushdown effectiveness)
SELECT
    catalog_name,
    COUNT(*) AS query_count,
    AVG(physical_input_bytes / NULLIF(output_bytes, 0)) AS avg_reduction_ratio
FROM system.runtime.queries
WHERE created > CURRENT_TIMESTAMP - INTERVAL '1' HOUR
  AND state = 'FINISHED'
GROUP BY catalog_name
ORDER BY avg_reduction_ratio DESC;
```

---

## Cluster Resource Utilization

### Worker CPU Usage

```sql
-- Worker node CPU utilization
SELECT
    node_id,
    node_version,
    processors,
    process_cpu_load,
    system_cpu_load,
    heap_used / heap_available AS heap_utilization
FROM system.runtime.nodes
ORDER BY process_cpu_load DESC;
```

### Memory Pressure

```sql
-- Identify memory pressure events
SELECT
    query_id,
    query,
    total_memory_bytes / 1024 / 1024 / 1024 AS memory_gb,
    error_type,
    failure_message
FROM system.runtime.queries
WHERE error_type = 'INSUFFICIENT_RESOURCES'
  AND created > CURRENT_TIMESTAMP - INTERVAL '24' HOUR
ORDER BY created DESC;
```

---

## EXPLAIN ANALYZE

### Detailed Execution Plan

```sql
-- Detailed execution plan with actual runtime statistics
EXPLAIN ANALYZE
SELECT region, COUNT(*) FROM iceberg.gold.sales
WHERE sale_date >= DATE '2024-01-01'
GROUP BY region;
```

### Output Interpretation

**Look for:**
- **Actual row counts**: Compare to estimates (large variance = stale stats)
- **Operator costs**: CPU time and wall time per stage
- **Data shuffling**: Cross-node data movement (expensive)
- **Pushdown confirmation**: RemoteQuery operators indicate pushdown
- **Memory spills**: Indicates workers ran out of memory

**Example output:**
```
Fragment 0 [SINGLE]
    Output layout: [region, count]
    Output partitioning: SINGLE
    Stage Execution Strategy: UNGROUPED_EXECUTION
    - Aggregate(FINAL) => [region:varchar, count:bigint]
            CPU: 45ms, Blocked: 0ms, Output: 5 rows
        - RemoteSource[1] => [region:varchar, count:bigint]
                CPU: 2ms, Output: 5 rows

Fragment 1 [HASH]
    Output layout: [region, count]
    Output partitioning: HASH [region]
    - Aggregate(PARTIAL) => [region:varchar, count:bigint]
            CPU: 1.2s, Output: 5 rows
        - ScanFilterProject[table = iceberg:gold:sales]
                Estimates: {rows: 1000000, cpu: ?, memory: ?, network: ?}
                CPU: 8.5s, Input: 1000000 rows, Output: 150000 rows
                filterPredicate := ("sale_date" >= DATE '2024-01-01')
```

**Analysis:**
- Partition pruning worked (150k rows vs 1M estimate)
- Filter pushed to Iceberg (filterPredicate in ScanFilterProject)
- Most time spent reading (8.5s CPU in Scan)
- Aggregation fast (1.2s for 150k → 5 rows)

---

## Alerting Queries

### High Queue Time Alert

```sql
-- Alert if average queue time > 5 seconds
SELECT
    AVG(queued_time_ms) / 1000 AS avg_queue_sec
FROM system.runtime.queries
WHERE created > CURRENT_TIMESTAMP - INTERVAL '5' MINUTE
  AND state IN ('RUNNING', 'FINISHED')
HAVING AVG(queued_time_ms) > 5000;
```

### Failed Query Spike Alert

```sql
-- Alert if > 10 failures in last 15 minutes
SELECT
    COUNT(*) AS failure_count,
    error_type
FROM system.runtime.queries
WHERE state = 'FAILED'
  AND created > CURRENT_TIMESTAMP - INTERVAL '15' MINUTE
GROUP BY error_type
HAVING COUNT(*) > 10;
```

### Memory Pressure Alert

```sql
-- Alert if any worker heap utilization > 90%
SELECT
    node_id,
    heap_used / heap_available AS heap_utilization
FROM system.runtime.nodes
WHERE heap_used / heap_available > 0.9;
```

---

## Cache Performance (Warp Speed)

### Cache Hit Rate

```sql
-- Warp Speed cache effectiveness
SELECT
    catalog_name,
    SUM(CASE WHEN cache_hit THEN 1 ELSE 0 END) AS cache_hits,
    COUNT(*) AS total_reads,
    SUM(CASE WHEN cache_hit THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS hit_rate_pct
FROM system.runtime.cache_stats
WHERE created > CURRENT_TIMESTAMP - INTERVAL '1' HOUR
GROUP BY catalog_name
ORDER BY hit_rate_pct DESC;
```

### Cache Size Utilization

```sql
-- Cache storage usage
SELECT
    node_id,
    cache_size_bytes / 1024 / 1024 / 1024 AS cache_gb,
    cache_max_size_bytes / 1024 / 1024 / 1024 AS cache_max_gb,
    cache_size_bytes * 100.0 / cache_max_size_bytes AS utilization_pct
FROM system.runtime.cache_storage
ORDER BY utilization_pct DESC;
```

---

**Last Updated:** 2026-02-16
