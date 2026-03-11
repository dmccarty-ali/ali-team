# ali-snowflake-data-engineer - Snowpipe Streaming API Reference

## Snowpipe Streaming API Code

Real-time row-level ingestion without staging files:

```java
// Java SDK example
import com.snowflake.ingest.streaming.*;

// Create client
SnowflakeStreamingIngestClient client = SnowflakeStreamingIngestClientFactory
    .builder("MY_CLIENT")
    .setProperties(props)
    .build();

// Open channel to table
SnowflakeStreamingIngestChannel channel = client.openChannel(
    OpenChannelRequest.builder("MY_CHANNEL")
        .setDBName("MY_DB")
        .setSchemaName("MY_SCHEMA")
        .setTableName("MY_TABLE")
        .setOnErrorOption(OpenChannelRequest.OnErrorOption.CONTINUE)
        .build()
);

// Insert rows
Map<String, Object> row = new HashMap<>();
row.put("col1", "value1");
row.put("col2", 123);
InsertValidationResponse response = channel.insertRow(row, "offset_token_1");

// Check for errors
if (response.hasErrors()) {
    for (InsertValidationResponse.InsertError error : response.getInsertErrors()) {
        System.err.println("Error: " + error.getMessage());
    }
}

// Close when done
channel.close().get();
client.close();
```

**Key Characteristics:**
- Sub-second latency (vs. Snowpipe's minute-level)
- No file staging required
- Exactly-once semantics with offset tokens
- Automatic micro-batch optimization
- Supports schema evolution

## Kafka Connector Configuration

```properties
# snowflake-kafka-connector configuration
name=snowflake_connector
connector.class=com.snowflake.kafka.connector.SnowflakeSinkConnector
tasks.max=8
topics=my_topic
snowflake.url.name=account.snowflakecomputing.com
snowflake.user.name=kafka_user
snowflake.private.key=...
snowflake.database.name=MY_DB
snowflake.schema.name=MY_SCHEMA
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=com.snowflake.kafka.connector.records.SnowflakeJsonConverter

# Ingestion method: snowpipe or snowpipe_streaming
snowflake.ingestion.method=SNOWPIPE_STREAMING

# Buffer settings
buffer.count.records=10000
buffer.flush.time=60
buffer.size.bytes=5000000
```

## Spark Connector Code

```python
# PySpark with Snowflake connector
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("SnowflakeIntegration") \
    .config("spark.jars.packages", "net.snowflake:spark-snowflake_2.12:2.12.0-spark_3.3") \
    .getOrCreate()

# Snowflake connection options
sfOptions = {
    "sfURL": "account.snowflakecomputing.com",
    "sfUser": "user",
    "sfPassword": "password",
    "sfDatabase": "MY_DB",
    "sfSchema": "MY_SCHEMA",
    "sfWarehouse": "MY_WH"
}

# Read from Snowflake
df = spark.read \
    .format("snowflake") \
    .options(**sfOptions) \
    .option("query", "SELECT * FROM my_table WHERE date > '2024-01-01'") \
    .load()

# Write to Snowflake
df.write \
    .format("snowflake") \
    .options(**sfOptions) \
    .option("dbtable", "target_table") \
    .mode("append") \
    .save()
```
