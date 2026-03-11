# ali-apache-iceberg - Hidden Partitioning Examples Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg hidden partitioning examples" or "I need detailed partition transform examples"

---

## Create Table with Hidden Partitioning

### Time-Series Data (Days)

```sql
-- Spark SQL
CREATE TABLE my_catalog.bronze_layer.cdc_orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_timestamp TIMESTAMP,
    region STRING,
    total_amount DECIMAL(12,2)
)
USING iceberg
PARTITIONED BY (days(order_timestamp), region);

-- Users query naturally:
SELECT * FROM cdc_orders
WHERE order_timestamp BETWEEN '2026-01-01' AND '2026-01-31'
  AND region = 'US-WEST';

-- Iceberg prunes to day-level partitions automatically
```

### High-Volume Streaming (Hours)

```sql
-- Hourly partitioning for high-frequency events
CREATE TABLE sensor_events (
    sensor_id STRING,
    event_timestamp TIMESTAMP,
    temperature DOUBLE,
    humidity DOUBLE
)
USING iceberg
PARTITIONED BY (hours(event_timestamp), bucket(16, sensor_id));

-- Query last 3 hours
SELECT * FROM sensor_events
WHERE event_timestamp > current_timestamp() - INTERVAL 3 HOURS
  AND sensor_id = 'SENSOR-12345';
```

### Hash Bucketing for Even Distribution

```sql
-- Prevent skew with bucket partitioning
CREATE TABLE user_events (
    user_id BIGINT,
    event_type STRING,
    event_timestamp TIMESTAMP,
    payload STRING
)
USING iceberg
PARTITIONED BY (days(event_timestamp), bucket(32, user_id));

-- Benefits:
-- - Even distribution across 32 buckets
-- - No hot partitions from popular users
-- - Efficient point lookups by user_id
```

### String Truncation

```sql
-- Partition by prefix (e.g., first 2 chars of country code)
CREATE TABLE global_transactions (
    transaction_id STRING,
    country_code STRING,  -- "US", "UK", "FR", etc.
    amount DECIMAL(12,2),
    transaction_timestamp TIMESTAMP
)
USING iceberg
PARTITIONED BY (truncate(2, country_code), months(transaction_timestamp));

-- Prunes to specific country prefix
SELECT * FROM global_transactions
WHERE country_code = 'US'
  AND transaction_timestamp >= '2026-01-01';
```

---

## Partition Evolution Examples

### Evolve Day → Month Partitioning

```sql
-- Initial partitioning by day
CREATE TABLE events (
    event_id BIGINT,
    event_timestamp TIMESTAMP,
    data STRING
)
USING iceberg
PARTITIONED BY (days(event_timestamp));

-- After 1 year, data accumulates - evolve to month partitioning
ALTER TABLE events
REPLACE PARTITION FIELD days(event_timestamp)
WITH months(event_timestamp);

-- Result:
-- - Old data (year 1): partitioned by day (365 partitions)
-- - New data (year 2+): partitioned by month (12 partitions/year)
-- - No data rewrite required!
-- - Queries work seamlessly across both schemes
```

### Add New Partition Column

```sql
-- Initially no partitioning
CREATE TABLE logs (
    log_timestamp TIMESTAMP,
    log_level STRING,
    message STRING
)
USING iceberg;

-- Add partitioning later
ALTER TABLE logs
ADD PARTITION FIELD days(log_timestamp);

-- New data partitioned, old data remains unpartitioned
-- Both readable in queries
```

### Remove Partition Column

```sql
-- Initially partitioned by region
CREATE TABLE sales (
    sale_id BIGINT,
    region STRING,
    amount DECIMAL(12,2),
    sale_timestamp TIMESTAMP
)
USING iceberg
PARTITIONED BY (region, months(sale_timestamp));

-- Region no longer useful - remove partition
ALTER TABLE sales
DROP PARTITION FIELD region;

-- Future data not partitioned by region
-- Old data still organized by region
```

---

## Multi-Level Partitioning Examples

### Time + Category

```sql
CREATE TABLE orders (
    order_id BIGINT,
    order_timestamp TIMESTAMP,
    order_type STRING,  -- "retail", "wholesale", "online"
    customer_id BIGINT,
    amount DECIMAL(12,2)
)
USING iceberg
PARTITIONED BY (days(order_timestamp), order_type);

-- Efficient queries on both dimensions
SELECT * FROM orders
WHERE order_timestamp = '2026-01-15'
  AND order_type = 'retail';
```

### Time + Geography + Hash

```sql
CREATE TABLE clickstream (
    event_id STRING,
    user_id BIGINT,
    event_timestamp TIMESTAMP,
    country_code STRING,
    event_type STRING
)
USING iceberg
PARTITIONED BY (
    hours(event_timestamp),    -- Time locality
    country_code,              -- Geographic locality
    bucket(16, user_id)        -- Even distribution
);

-- Query benefits from all three partition levels
SELECT * FROM clickstream
WHERE event_timestamp > current_timestamp() - INTERVAL 6 HOURS
  AND country_code = 'US'
  AND user_id = 12345;
```

---

## Partition Transform Comparison

### Year vs Month vs Day vs Hour

```sql
-- Same table with different granularities
CREATE TABLE events_year (ts TIMESTAMP, data STRING)
PARTITIONED BY (years(ts));    -- ~1 partition/year (low cardinality)

CREATE TABLE events_month (ts TIMESTAMP, data STRING)
PARTITIONED BY (months(ts));   -- ~12 partitions/year

CREATE TABLE events_day (ts TIMESTAMP, data STRING)
PARTITIONED BY (days(ts));     -- ~365 partitions/year

CREATE TABLE events_hour (ts TIMESTAMP, data STRING)
PARTITIONED BY (hours(ts));    -- ~8,760 partitions/year (high cardinality)
```

**Choosing granularity:**
- **Year**: Archival data, rarely queried
- **Month**: Moderate-frequency queries (monthly reports)
- **Day**: Common for analytics workloads
- **Hour**: High-frequency streaming data

**Cardinality warning**: Too many partitions (10,000+) degrades performance.

---

## Identity Partitioning (Exact Value)

```sql
-- Partition by exact region value (low cardinality)
CREATE TABLE regional_data (
    data_id BIGINT,
    region STRING,  -- "US-EAST", "US-WEST", "EU-CENTRAL" (few values)
    data STRING
)
USING iceberg
PARTITIONED BY (identity(region));

-- Efficient when:
-- - Column has low cardinality (< 100 distinct values)
-- - Queries filter by exact value frequently
```

---

## Anti-Patterns: When NOT to Partition

### High Cardinality Identity

```sql
-- BAD: Partitioning by high-cardinality column
CREATE TABLE orders (
    order_id BIGINT,  -- Unique per row
    customer_id BIGINT,  -- Millions of distinct values
    order_timestamp TIMESTAMP
)
USING iceberg
PARTITIONED BY (identity(customer_id));  -- Creates millions of partitions!

-- GOOD: Use bucketing instead
PARTITIONED BY (days(order_timestamp), bucket(16, customer_id));
```

### Over-Partitioning

```sql
-- BAD: Too fine granularity for workload
CREATE TABLE monthly_reports (
    report_id BIGINT,
    report_timestamp TIMESTAMP,  -- Only 12 reports per year
    data STRING
)
USING iceberg
PARTITIONED BY (days(report_timestamp));  -- 365 mostly empty partitions

-- GOOD: Match granularity to data frequency
PARTITIONED BY (months(report_timestamp));  -- 12 partitions
```

---

## References

- [Iceberg Partition Transforms](https://iceberg.apache.org/docs/latest/partitioning/)
- [Partition Evolution Spec](https://iceberg.apache.org/spec/#partition-evolution)
- [Netflix Partitioning Strategy](https://netflixtechblog.com/how-netflix-uses-iceberg-for-big-data-analytics-8c9ffc6c7e92)
