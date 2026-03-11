# ali-apache-iceberg - CDC Implementation Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg CDC implementation details" or "I need Debezium/Kafka/Flink CDC examples"

---

## Debezium CDC Message Structure

### PostgreSQL CDC Message Example

```json
{
  "before": {
    "id": 123,
    "name": "Old Name",
    "email": "old@example.com",
    "status": "pending",
    "updated_at": 1705334400000
  },
  "after": {
    "id": 123,
    "name": "New Name",
    "email": "new@example.com",
    "status": "approved",
    "updated_at": 1705420800000
  },
  "source": {
    "version": "2.4.0.Final",
    "connector": "postgresql",
    "name": "dbserver1",
    "ts_ms": 1705420800123,
    "snapshot": "false",
    "db": "production",
    "schema": "public",
    "table": "users",
    "txId": 987654,
    "lsn": 33023768,
    "xmin": null
  },
  "op": "u",  // c=create, u=update, d=delete, r=read (snapshot)
  "ts_ms": 1705420800456,
  "transaction": null
}
```

### Operation Types

| Operation | Description | Contains |
|-----------|-------------|----------|
| `c` (create) | INSERT | `after` only, `before` = null |
| `u` (update) | UPDATE | Both `before` and `after` |
| `d` (delete) | DELETE | `before` only, `after` = null |
| `r` (read) | Initial snapshot | `after` only, `before` = null |
| `t` (truncate) | TRUNCATE TABLE | Neither `before` nor `after` |

---

## Flink CDC Ingestion

### Complete Flink CDC Pipeline

```java
import org.apache.flink.table.api.*;

public class FlinkCDCToIceberg {
    public static void main(String[] args) {
        // Create Flink streaming environment
        EnvironmentSettings settings = EnvironmentSettings
            .newInstance()
            .inStreamingMode()
            .build();
        TableEnvironment tEnv = TableEnvironment.create(settings);

        // Create Iceberg catalog
        tEnv.executeSql(
            "CREATE CATALOG my_catalog WITH (\n" +
            "  'type'='iceberg',\n" +
            "  'catalog-type'='hive',\n" +
            "  'uri'='thrift://metastore:9083',\n" +
            "  'warehouse'='s3://my-bucket/warehouse'\n" +
            ")"
        );
        tEnv.executeSql("USE CATALOG my_catalog");

        // Create Kafka CDC source
        tEnv.executeSql(
            "CREATE TABLE kafka_cdc_source (\n" +
            "  `before` ROW<\n" +
            "    id BIGINT,\n" +
            "    name STRING,\n" +
            "    email STRING,\n" +
            "    status STRING,\n" +
            "    updated_at TIMESTAMP(3)\n" +
            "  >,\n" +
            "  `after` ROW<\n" +
            "    id BIGINT,\n" +
            "    name STRING,\n" +
            "    email STRING,\n" +
            "    status STRING,\n" +
            "    updated_at TIMESTAMP(3)\n" +
            "  >,\n" +
            "  `op` STRING,\n" +
            "  `ts_ms` BIGINT,\n" +
            "  `source` ROW<\n" +
            "    table STRING,\n" +
            "    lsn BIGINT,\n" +
            "    txId BIGINT\n" +
            "  >\n" +
            ") WITH (\n" +
            "  'connector' = 'kafka',\n" +
            "  'topic' = 'dbserver1.public.users',\n" +
            "  'properties.bootstrap.servers' = 'kafka:9092',\n" +
            "  'properties.group.id' = 'flink-cdc-consumer',\n" +
            "  'scan.startup.mode' = 'earliest-offset',\n" +
            "  'format' = 'debezium-json'\n" +
            ")"
        );

        // Create Iceberg sink table
        tEnv.executeSql(
            "CREATE TABLE iceberg_users (\n" +
            "  id BIGINT,\n" +
            "  name STRING,\n" +
            "  email STRING,\n" +
            "  status STRING,\n" +
            "  updated_at TIMESTAMP(3),\n" +
            "  cdc_timestamp TIMESTAMP(3),\n" +
            "  PRIMARY KEY (id) NOT ENFORCED\n" +
            ") PARTITIONED BY (HOUR(updated_at))\n" +
            "WITH (\n" +
            "  'connector' = 'iceberg',\n" +
            "  'catalog-name' = 'my_catalog',\n" +
            "  'database-name' = 'bronze_layer',\n" +
            "  'table-name' = 'users',\n" +
            "  'write.upsert.enabled' = 'true'\n" +
            ")"
        );

        // Transform and upsert CDC to Iceberg
        tEnv.executeSql(
            "INSERT INTO iceberg_users\n" +
            "SELECT\n" +
            "  COALESCE(`after`.id, `before`.id) AS id,\n" +
            "  `after`.name,\n" +
            "  `after`.email,\n" +
            "  `after`.status,\n" +
            "  `after`.updated_at,\n" +
            "  TO_TIMESTAMP(FROM_UNIXTIME(ts_ms / 1000)) AS cdc_timestamp\n" +
            "FROM kafka_cdc_source\n" +
            "WHERE op IN ('c', 'u', 'r')  -- INSERT/UPDATE operations only\n"
        );
    }
}
```

### Flink CDC with Delete Handling

```java
// Handle deletes explicitly
tEnv.executeSql(
    "CREATE TABLE iceberg_users_with_deletes (\n" +
    "  id BIGINT,\n" +
    "  name STRING,\n" +
    "  email STRING,\n" +
    "  status STRING,\n" +
    "  is_deleted BOOLEAN,\n" +  // Soft delete flag
    "  deleted_at TIMESTAMP(3),\n" +
    "  PRIMARY KEY (id) NOT ENFORCED\n" +
    ") WITH (\n" +
    "  'connector' = 'iceberg',\n" +
    "  'write.upsert.enabled' = 'true'\n" +
    ")"
);

tEnv.executeSql(
    "INSERT INTO iceberg_users_with_deletes\n" +
    "SELECT\n" +
    "  COALESCE(`after`.id, `before`.id) AS id,\n" +
    "  `after`.name,\n" +
    "  `after`.email,\n" +
    "  `after`.status,\n" +
    "  CASE WHEN op = 'd' THEN TRUE ELSE FALSE END AS is_deleted,\n" +
    "  CASE WHEN op = 'd' THEN TO_TIMESTAMP(FROM_UNIXTIME(ts_ms / 1000)) ELSE NULL END AS deleted_at\n" +
    "FROM kafka_cdc_source\n"
);
```

---

## Spark Structured Streaming CDC

### Spark CDC with MERGE

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json, schema_of_json, to_timestamp, from_unixtime

spark = SparkSession.builder \
    .appName("CDC to Iceberg") \
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .getOrCreate()

# Debezium CDC schema
cdc_schema = """
struct<
  before: struct<id: bigint, name: string, email: string>,
  after: struct<id: bigint, name: string, email: string>,
  op: string,
  ts_ms: bigint
>
"""

# Read from Kafka
kafka_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "dbserver1.public.users") \
    .option("startingOffsets", "earliest") \
    .load()

# Parse Debezium JSON
cdc_df = kafka_df.selectExpr("CAST(value AS STRING) as json") \
    .select(from_json(col("json"), cdc_schema).alias("data")) \
    .select("data.*") \
    .select(
        col("after.id").alias("id"),
        col("after.name").alias("name"),
        col("after.email").alias("email"),
        to_timestamp(from_unixtime(col("ts_ms") / 1000)).alias("cdc_timestamp"),
        col("op")
    ) \
    .filter(col("op").isin("c", "u", "r"))  # INSERT/UPDATE only

# Upsert to Iceberg using foreachBatch
def upsert_to_iceberg(batch_df, batch_id):
    batch_df.createOrReplaceTempView("updates")

    batch_df.sparkSession.sql("""
        MERGE INTO my_catalog.bronze_layer.users target
        USING updates source
        ON target.id = source.id
        WHEN MATCHED THEN
            UPDATE SET
                target.name = source.name,
                target.email = source.email,
                target.cdc_timestamp = source.cdc_timestamp
        WHEN NOT MATCHED THEN
            INSERT (id, name, email, cdc_timestamp)
            VALUES (source.id, source.name, source.email, source.cdc_timestamp)
    """)

# Write stream with MERGE upsert
cdc_df.writeStream \
    .foreachBatch(upsert_to_iceberg) \
    .option("checkpointLocation", "s3://checkpoints/users") \
    .start() \
    .awaitTermination()
```

### Spark CDC with Append-Only (Slowly Changing Dimension Type 2)

```python
# SCD Type 2: Keep full history of changes
kafka_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "dbserver1.public.users") \
    .load()

cdc_df = kafka_df.selectExpr("CAST(value AS STRING) as json") \
    .select(from_json(col("json"), cdc_schema).alias("data")) \
    .select(
        col("data.after.id").alias("id"),
        col("data.after.name").alias("name"),
        col("data.after.email").alias("email"),
        to_timestamp(from_unixtime(col("data.ts_ms") / 1000)).alias("valid_from"),
        col("data.op")
    ) \
    .filter(col("op").isin("c", "u", "r"))

# Append all changes (every version is a new row)
cdc_df.writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", "s3://checkpoints/users_scd2") \
    .option("path", "s3://warehouse/bronze_layer/users_history") \
    .toTable("my_catalog.bronze_layer.users_history")
```

---

## Exactly-Once Semantics

### Kafka Offset Management with Iceberg

```python
# Iceberg + Kafka provides exactly-once processing via checkpointing
spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "orders") \
    .option("startingOffsets", "earliest") \
    .load() \
    .writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("checkpointLocation", "s3://checkpoints/orders") \
    .option("fanout-enabled", "true") \
    .toTable("my_catalog.bronze_layer.orders")

# How it works:
# 1. Spark writes data to Iceberg
# 2. Iceberg creates snapshot
# 3. Spark commits Kafka offsets + Iceberg snapshot ID to checkpoint
# 4. On restart, Spark resumes from last committed offset
# 5. Duplicate processing prevented by checkpoint coordination
```

### Idempotency with Deduplication

```python
# Add deduplication window to handle duplicates
from pyspark.sql.functions import window

cdc_df_deduplicated = cdc_df \
    .withWatermark("cdc_timestamp", "10 minutes") \
    .dropDuplicates(["id", "cdc_timestamp"])

cdc_df_deduplicated.writeStream \
    .foreachBatch(upsert_to_iceberg) \
    .option("checkpointLocation", "s3://checkpoints/users_dedup") \
    .start()
```

---

## Multi-Table CDC Pattern

### Parallel CDC Ingestion

```python
# Ingest multiple tables concurrently
tables = [
    ("dbserver1.public.users", "bronze_layer.users"),
    ("dbserver1.public.orders", "bronze_layer.orders"),
    ("dbserver1.public.products", "bronze_layer.products")
]

for kafka_topic, iceberg_table in tables:
    kafka_df = spark.readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "kafka:9092") \
        .option("subscribe", kafka_topic) \
        .load()

    # Process and write to Iceberg
    # (transformation logic per table)

    kafka_df.writeStream \
        .foreachBatch(lambda df, id: upsert_to_iceberg(df, id, iceberg_table)) \
        .option("checkpointLocation", f"s3://checkpoints/{iceberg_table}") \
        .start()
```

---

## Schema Registry Integration

### Avro Schema Registry with Debezium

```python
# Use Confluent Schema Registry for Avro schemas
from pyspark.sql.avro.functions import from_avro

# Read Avro with schema registry
kafka_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "dbserver1.public.users") \
    .option("kafka.schema.registry.url", "http://schema-registry:8081") \
    .load()

# Deserialize Avro
cdc_df = kafka_df.select(
    from_avro(col("value"), "user-value").alias("data")
).select("data.*")
```

---

## CDC Monitoring

### Lag Monitoring

```python
# Monitor Kafka consumer lag
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'dbserver1.public.users',
    bootstrap_servers='kafka:9092',
    group_id='flink-cdc-consumer'
)

end_offsets = consumer.end_offsets(consumer.assignment())
current_offsets = consumer.committed(consumer.assignment())

for partition, end_offset in end_offsets.items():
    current = current_offsets.get(partition, 0)
    lag = end_offset - current
    if lag > 10000:
        print(f"WARNING: High lag on partition {partition}: {lag} messages")
```

### CDC Pipeline Health Check

```sql
-- Check CDC freshness in Iceberg
SELECT
    MAX(cdc_timestamp) AS latest_cdc,
    current_timestamp() - MAX(cdc_timestamp) AS lag_seconds
FROM my_catalog.bronze_layer.users;

-- Alert if lag > 5 minutes
```

---

## Troubleshooting CDC

### Issue: Duplicate Records

**Cause**: Kafka reprocessing without deduplication

**Solution**: Add watermark and dropDuplicates

```python
cdc_df_deduplicated = cdc_df \
    .withWatermark("cdc_timestamp", "10 minutes") \
    .dropDuplicates(["id", "cdc_timestamp"])
```

---

### Issue: High Latency

**Cause**: Copy-on-write mode rewriting entire files

**Solution**: Use merge-on-read

```sql
ALTER TABLE users SET TBLPROPERTIES (
    'write.update.mode' = 'merge-on-read',
    'write.delete.mode' = 'merge-on-read'
);
```

---

### Issue: Schema Evolution Failures

**Cause**: Debezium schema change not compatible

**Solution**: Use Avro with Schema Registry for automatic schema evolution

```python
# Avro handles schema evolution automatically
kafka_df = spark.readStream \
    .format("kafka") \
    .option("kafka.schema.registry.url", "http://schema-registry:8081") \
    .load()
```

---

## References

- [Debezium Documentation](https://debezium.io/documentation/)
- [Flink CDC Connector](https://nightlies.apache.org/flink/flink-cdc-docs-stable/)
- [Spark Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [Iceberg Streaming Writes](https://iceberg.apache.org/docs/latest/spark-structured-streaming/)
