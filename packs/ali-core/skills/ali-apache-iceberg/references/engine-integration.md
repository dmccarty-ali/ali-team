# ali-apache-iceberg - Engine Integration Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg engine integration details" or "I need Spark/Flink/Trino/Snowflake examples"

---

## Spark Integration

### Spark Session Configuration

```python
# Spark 3.3+ with Iceberg
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("IcebergApp") \
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.my_catalog", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.my_catalog.type", "hive") \
    .config("spark.sql.catalog.my_catalog.uri", "thrift://metastore:9083") \
    .config("spark.sql.catalog.my_catalog.warehouse", "s3://my-bucket/warehouse") \
    .getOrCreate()
```

### Spark DataFrame Read/Write

```python
# Read Iceberg table
df = spark.table("my_catalog.bronze_layer.orders")

# Write to Iceberg table
df.writeTo("my_catalog.bronze_layer.orders").append()

# Overwrite partitions
df.writeTo("my_catalog.bronze_layer.orders").overwritePartitions()

# Create table if not exists
df.writeTo("my_catalog.bronze_layer.new_table") \
    .tableProperty("write.format.default", "parquet") \
    .partitionedBy("order_date") \
    .createOrReplace()
```

### Spark SQL Operations

```python
# Create table
spark.sql("""
    CREATE TABLE my_catalog.bronze_layer.events (
        event_id BIGINT,
        event_timestamp TIMESTAMP,
        event_type STRING
    )
    USING iceberg
    PARTITIONED BY (days(event_timestamp))
""")

# Time travel
spark.sql("""
    SELECT * FROM orders
    FOR SYSTEM_TIME AS OF '2026-01-15 10:00:00'
""").show()

# Metadata queries
spark.sql("SELECT * FROM orders.snapshots").show()
spark.sql("SELECT * FROM orders.files").show()
```

---

## Flink Integration

### Flink SQL Catalog Setup

```java
// Flink SQL with Iceberg
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableEnvironment;

TableEnvironment tEnv = TableEnvironment.create(
    EnvironmentSettings.inStreamingMode()
);

tEnv.executeSql(
    "CREATE CATALOG my_catalog WITH (\n" +
    "  'type'='iceberg',\n" +
    "  'catalog-type'='hive',\n" +
    "  'uri'='thrift://metastore:9083',\n" +
    "  'warehouse'='s3://my-bucket/warehouse'\n" +
    ")"
);

tEnv.executeSql("USE CATALOG my_catalog");
tEnv.executeSql("USE bronze_layer");
```

### Flink Streaming Write

```java
// Streaming write to Iceberg
tEnv.executeSql(
    "INSERT INTO orders\n" +
    "SELECT * FROM kafka_source"
);
```

### Flink CDC to Iceberg

```java
// Debezium CDC → Iceberg upsert
tEnv.executeSql(
    "CREATE TABLE kafka_cdc (\n" +
    "  `before` ROW<id BIGINT, name STRING>,\n" +
    "  `after` ROW<id BIGINT, name STRING>,\n" +
    "  `op` STRING\n" +
    ") WITH (\n" +
    "  'connector' = 'kafka',\n" +
    "  'topic' = 'dbserver.public.users',\n" +
    "  'properties.bootstrap.servers' = 'kafka:9092',\n" +
    "  'format' = 'debezium-json'\n" +
    ")"
);

tEnv.executeSql(
    "CREATE TABLE users (\n" +
    "  id BIGINT,\n" +
    "  name STRING,\n" +
    "  PRIMARY KEY (id) NOT ENFORCED\n" +
    ") WITH (\n" +
    "  'connector' = 'iceberg',\n" +
    "  'catalog-name' = 'my_catalog',\n" +
    "  'database-name' = 'bronze_layer',\n" +
    "  'table-name' = 'users'\n" +
    ")"
);

// Upsert from CDC stream
tEnv.executeSql("INSERT INTO users SELECT COALESCE(`after`.id, `before`.id), `after`.name FROM kafka_cdc WHERE op IN ('c', 'u')");
```

### Flink Configuration Options

```java
// Configure Flink Iceberg write
tEnv.getConfig().getConfiguration().setString(
    "table.exec.iceberg.write.upsert-mode", "merge-on-read"
);
```

---

## Trino/Starburst Integration

### Trino Catalog Configuration

```properties
# catalog/iceberg.properties
connector.name=iceberg
iceberg.catalog.type=hive_metastore
hive.metastore.uri=thrift://metastore:9083
hive.s3.path-style-access=true
hive.s3.endpoint=https://s3.us-east-1.amazonaws.com
```

### Trino SQL Queries

```sql
-- Query Iceberg tables
SELECT * FROM iceberg.bronze_layer.orders
WHERE order_date = DATE '2026-01-15';

-- Time travel
SELECT * FROM iceberg.bronze_layer.orders
FOR TIMESTAMP AS OF TIMESTAMP '2026-01-15 10:00:00';

-- Metadata queries
SELECT * FROM iceberg.bronze_layer."orders$snapshots";
SELECT * FROM iceberg.bronze_layer."orders$files";
SELECT * FROM iceberg.bronze_layer."orders$manifests";
```

### Trino Write Operations

```sql
-- Insert
INSERT INTO iceberg.bronze_layer.events
SELECT * FROM staging.events;

-- MERGE (Trino 400+)
MERGE INTO iceberg.bronze_layer.customers target
USING staging.customers source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN UPDATE SET name = source.name
WHEN NOT MATCHED THEN INSERT VALUES (source.customer_id, source.name);
```

### Starburst-Specific Features

```sql
-- Materialized views on Iceberg
CREATE MATERIALIZED VIEW order_summary AS
SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total
FROM iceberg.bronze_layer.orders
GROUP BY customer_id;

-- Scheduled refresh
ALTER MATERIALIZED VIEW order_summary SET PROPERTIES refresh_interval = '1h';
```

---

## Snowflake Integration

### Snowflake External Table

```sql
-- Create external volume for Iceberg
CREATE EXTERNAL VOLUME iceberg_volume
   STORAGE_LOCATIONS = (
       (
           NAME = 's3_location'
           STORAGE_PROVIDER = 'S3'
           STORAGE_BASE_URL = 's3://my-bucket/warehouse/'
           STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-role'
       )
   );

-- Create Iceberg external table
CREATE ICEBERG TABLE bronze_orders
  EXTERNAL_VOLUME = 'iceberg_volume'
  CATALOG = 'my_glue_catalog'
  CATALOG_TABLE_NAME = 'bronze_layer.orders'
  CATALOG_INTEGRATION = glue_integration
  AUTO_REFRESH = TRUE
  REFRESH_INTERVAL_SECONDS = 300;

-- Query Iceberg table from Snowflake
SELECT * FROM bronze_orders WHERE order_date = '2026-01-15';
```

### Snowflake Catalog Integration

```sql
-- Create Glue catalog integration
CREATE CATALOG INTEGRATION glue_integration
  CATALOG_SOURCE = GLUE
  TABLE_FORMAT = ICEBERG
  GLUE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-glue-role'
  GLUE_CATALOG_ID = '123456789012'
  GLUE_REGION = 'us-east-1'
  ENABLED = TRUE;
```

---

## AWS Athena Integration

### Athena Iceberg Table

```sql
-- Create Iceberg table in Athena
CREATE TABLE bronze_layer.orders (
    order_id BIGINT,
    order_date DATE,
    amount DECIMAL(12,2)
)
LOCATION 's3://my-bucket/warehouse/bronze_layer/orders/'
TBLPROPERTIES (
    'table_type' = 'ICEBERG',
    'format' = 'parquet',
    'write_compression' = 'zstd'
);

-- Query
SELECT * FROM bronze_layer.orders
WHERE order_date = DATE '2026-01-15';
```

### Athena Time Travel

```sql
-- Time travel in Athena (Athena v3)
SELECT * FROM bronze_layer.orders
FOR SYSTEM_TIME AS OF TIMESTAMP '2026-01-15 10:00:00';

-- Query specific snapshot
SELECT * FROM bronze_layer.orders
FOR SYSTEM_VERSION AS OF 1234567890;
```

---

## Presto Integration

### Presto Catalog Configuration

```properties
# catalog/iceberg.properties
connector.name=iceberg
hive.metastore.uri=thrift://metastore:9083
hive.s3.path-style-access=true
```

### Presto Queries

```sql
-- Standard queries
SELECT * FROM iceberg.bronze_layer.orders;

-- Metadata
SELECT * FROM iceberg.bronze_layer."orders$snapshots";
```

---

## Dremio Integration

### Dremio Iceberg Source

```sql
-- Add Iceberg source in Dremio UI
-- Source Type: Amazon S3
-- Enable Iceberg format detection

-- Query Iceberg tables
SELECT * FROM s3.bronze_layer.orders;
```

---

## DuckDB Integration

### DuckDB with Iceberg Extension

```sql
-- Install Iceberg extension
INSTALL iceberg;
LOAD iceberg;

-- Query Iceberg table
SELECT * FROM iceberg_scan('s3://my-bucket/warehouse/bronze_layer/orders/metadata/v1.metadata.json');
```

---

## PyIceberg (Python Native)

### PyIceberg Usage

```python
from pyiceberg.catalog import load_catalog

# Load catalog
catalog = load_catalog({
    'type': 'hive',
    'uri': 'thrift://metastore:9083',
    'warehouse': 's3://my-bucket/warehouse'
})

# List tables
tables = catalog.list_tables('bronze_layer')

# Load table
table = catalog.load_table('bronze_layer.orders')

# Scan table
scan = table.scan()
for batch in scan.to_arrow():
    print(batch)

# Time travel
snapshot = table.snapshot_by_id(1234567890)
scan = table.scan(snapshot_id=snapshot.snapshot_id)
```

---

## Engine Comparison

| Feature | Spark | Flink | Trino | Snowflake | Athena | Presto |
|---------|-------|-------|-------|-----------|--------|--------|
| **Read** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Write** | ✅ | ✅ | ✅ | ❌ | ✅ | ⚠️ Limited |
| **MERGE** | ✅ | ✅ | ✅ | ❌ | ✅ | ⚠️ Partial |
| **Time Travel** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Streaming** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **CDC** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Compaction** | ✅ | ⚠️ Manual | ⚠️ Manual | ❌ | ⚠️ Manual | ❌ |

---

## Multi-Engine Access Pattern

### Recommended Architecture

```
Write Path:
  PostgreSQL → Debezium → Kafka → Flink → Iceberg (Bronze)

Query Paths:
  - Trino/Starburst: Interactive analytics
  - Spark: Batch transformations, compaction
  - Athena: Ad-hoc queries
  - Snowflake: BI tool queries (read-only)
```

### Catalog Synchronization

```python
# Ensure all engines see latest snapshot
# Iceberg handles this automatically via metadata.json pointer

# No cache invalidation needed - each engine reads current metadata
```

---

## References

- [Spark Iceberg Documentation](https://iceberg.apache.org/docs/latest/spark-configuration/)
- [Flink Iceberg Connector](https://iceberg.apache.org/docs/latest/flink/)
- [Trino Iceberg Connector](https://trino.io/docs/current/connector/iceberg.html)
- [Athena Iceberg Support](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
- [Snowflake Iceberg Tables](https://docs.snowflake.com/en/user-guide/tables-iceberg.html)
