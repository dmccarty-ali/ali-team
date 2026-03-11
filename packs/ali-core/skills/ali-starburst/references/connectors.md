# ali-starburst - Connectors Reference

Detailed connector configurations, examples, and connector-specific patterns.

---

## Iceberg Connector (First-Class)

Starburst has the most complete Iceberg support in the industry.

### SQL Examples

```sql
-- Time travel queries
SELECT * FROM iceberg.bronze.events
FOR VERSION AS OF 12345;  -- Snapshot ID

SELECT * FROM iceberg.bronze.events
FOR TIMESTAMP AS OF TIMESTAMP '2024-01-15 10:00:00';

-- Schema evolution queries (works transparently)
SELECT new_column FROM iceberg.silver.customers;
-- Reads rows without new_column as NULL

-- Partition evolution (invisible to queries)
-- Table initially partitioned by date, evolved to date + region
SELECT * FROM iceberg.gold.sales WHERE region = 'US';
-- Query optimizer uses current partitioning

-- Metadata queries
SELECT * FROM iceberg.bronze.events$snapshots;
SELECT * FROM iceberg.bronze.events$files;
SELECT * FROM iceberg.bronze.events$manifests;
```

### Iceberg REST Catalog Configuration

```properties
# catalog/iceberg.properties
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=https://catalog.example.com
iceberg.rest-catalog.warehouse=s3://lake-warehouse/
iceberg.rest-catalog.security=OAUTH2

# Authentication
iceberg.rest-catalog.oauth2.credential=<token>
```

**Why REST Catalog:**
- Centralized metadata management
- Multi-engine support (Starburst, Spark, Flink)
- Atomic table operations
- S3 object locking not required

---

## Hive Connector

### Configuration

```properties
# catalog/hive.properties
connector.name=hive
hive.metastore.uri=thrift://metastore:9083
hive.s3.aws-access-key=<key>
hive.s3.aws-secret-key=<secret>
hive.s3.endpoint=s3.amazonaws.com

# Performance
hive.max-partitions-per-scan=10000
hive.metastore-timeout=30s
```

### Limitations

- No UPDATE/DELETE (append-only)
- Partition pruning requires explicit partition filters
- Schema evolution limited

---

## JDBC Connectors

### PostgreSQL Configuration

```properties
# catalog/postgresql.properties
connector.name=postgresql
connection-url=jdbc:postgresql://hostname:5432/database
connection-user=readonly_user
connection-password=<password>

# Pushdown configuration
jdbc.pushdown-enabled=true
```

### Pushdown Capabilities

| Operation | PostgreSQL | SQL Server | Oracle | MySQL |
|-----------|-----------|-----------|--------|-------|
| Filter (WHERE) | Full | Full | Full | Full |
| Projection (SELECT) | Full | Full | Full | Full |
| Aggregation (GROUP BY) | Full | Full | Full | Full |
| JOIN | Limited | Limited | Limited | Limited |
| LIMIT | Yes | Yes | Yes | Yes |
| ORDER BY | Partial | Partial | Partial | Partial |

### JDBC Connector Patterns

```sql
-- Good: Filter pushed to source
SELECT * FROM postgresql.public.users
WHERE created_at > DATE '2024-01-01';
-- Trino sends: SELECT * FROM users WHERE created_at > '2024-01-01'

-- Good: Aggregation pushed to source
SELECT status, COUNT(*) FROM postgresql.public.orders
GROUP BY status;
-- Trino sends: SELECT status, COUNT(*) FROM orders GROUP BY status

-- Bad: JOIN not pushed, data pulled to Trino
SELECT u.name, o.total
FROM postgresql.public.users u
JOIN postgresql.public.orders o ON u.id = o.user_id;
-- Trino pulls both full tables, joins locally (expensive)
```

---

## Snowflake Connector

### Configuration

```properties
# catalog/snowflake.properties
connector.name=snowflake
connection-url=jdbc:snowflake://account.snowflakecomputing.com
connection-user=user
connection-password=<password>
snowflake.account=account
snowflake.database=DATABASE
snowflake.role=ROLE
snowflake.warehouse=WAREHOUSE
```

### Use Cases

- Join Snowflake warehouse with Iceberg lake
- Cross-warehouse analytics (Snowflake + Redshift)
- Migrate from Snowflake to Iceberg incrementally

### Pushdown Capabilities

- Full pushdown for filters, aggregations, LIMIT
- JOIN pushdown to Snowflake when all tables in same catalog

---

## Warp Speed (Caching)

### How It Works

```
Query Request
     │
     ▼
Cache Hit? ──Yes──> Return cached data
     │
     No
     │
     ▼
Execute query against source
     │
     ▼
Store result in cache
     │
     ▼
Return data
```

### Configuration

```properties
# config.properties
# Enable Warp Speed on all workers
node-scheduler.include-coordinator=false

# Cache storage (S3, local SSD)
cache.base-directory=/mnt/cache
cache.disk-usage-percentage=80
cache.max-in-memory-cache-size=1GB

# Alluxio-based caching (for S3/HDFS)
hive.cache.enabled=true
hive.cache.location=alluxio://alluxio-master:19998/cache
```

### When Warp Speed Helps

| Scenario | Benefit |
|----------|---------|
| Slow source (RDBMS over WAN) | Cache eliminates network latency |
| Repeated queries (dashboards) | Second query instant |
| Large scans (fact tables) | Cache stores filtered results |
| Hot partitions | Frequently accessed partitions cached |

### Cache Invalidation

```sql
-- Manual cache invalidation
CALL system.flush_metadata_cache(
    schema_name => 'iceberg.bronze',
    table_name => 'events'
);

-- Automatic invalidation (Iceberg)
-- Cache invalidated on snapshot change (INSERT, UPDATE, DELETE)
```

---

**Last Updated:** 2026-02-16
