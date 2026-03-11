# ali-snowflake-data-engineer - Clustering Analysis Reference

## Detailed Clustering Information

```sql
-- Detailed clustering info
SELECT SYSTEM$CLUSTERING_INFORMATION('my_table', '(date_col, region)');

-- Returns:
-- {
--   "cluster_by_keys": "LINEAR(date_col, region)",
--   "total_partition_count": 1000,
--   "total_constant_partition_count": 50,  -- fully clustered
--   "average_overlaps": 2.5,               -- lower is better
--   "average_depth": 1.8,                  -- lower is better
--   "partition_depth_histogram": {...}
-- }

-- Reclustering (automatic, but can force)
ALTER TABLE my_table RECLUSTER;

-- Suspend/resume auto-clustering
ALTER TABLE my_table SUSPEND RECLUSTER;
ALTER TABLE my_table RESUME RECLUSTER;
```

## Multi-Column Clustering

```sql
-- Multi-column clustering
ALTER TABLE events CLUSTER BY (event_date, event_type);

-- Expression-based clustering
ALTER TABLE events CLUSTER BY (DATE_TRUNC('DAY', event_timestamp), region);
```

## Clustering Depth Check

```sql
-- Clustering depth (how well partitioned)
SELECT SYSTEM$CLUSTERING_DEPTH('my_table');
```
