# ali-data-architecture - Partitioning Strategies Reference

Table partitioning and clustering for performance.

---

## Partitioning Strategies

| Strategy | Use Case | Example |
|----------|----------|---------|
| **Time** | Time-series, append-heavy | `PARTITION BY RANGE (service_date)` |
| **Hash** | Even distribution, key lookups | `PARTITION BY HASH (claim_id)` |
| **List** | Known categories | `PARTITION BY LIST (region)` |

---

## Snowflake Clustering

```sql
-- Cluster on commonly filtered columns
ALTER TABLE EDW_MEDEDI.FACT_CLAIM
CLUSTER BY (service_date, payer_id);

-- Snowflake auto-maintains clustering
-- Check clustering depth
SELECT SYSTEM$CLUSTERING_INFORMATION('EDW_MEDEDI.FACT_CLAIM');

-- Lower depth = better clustering (0-4 typical)
```

---

## Benefits

- **Query performance**: Prune partitions, skip irrelevant data
- **Maintenance**: Drop old partitions easily
- **Parallelism**: Query partitions in parallel

---

## Recommendations

- Partition on frequently filtered columns (dates, regions)
- Snowflake: Use clustering instead of manual partitions
- Keep partitions balanced (avoid skew)
- Monitor clustering depth, reclustering costs
