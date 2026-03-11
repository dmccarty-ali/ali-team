# ali-apache-kafka - Kafka Connect Reference

## Overview

Kafka Connect is a framework for streaming data between Apache Kafka and other systems. It provides scalable, reliable data integration with minimal code.

---

## Debezium CDC Pattern

**Use case:** Capture database changes and publish to Kafka

```json
{
  "name": "debezium-sap-source",
  "config": {
    "connector.class": "io.debezium.connector.oracle.OracleConnector",
    "tasks.max": "1",
    "database.hostname": "sap-mdg-prod.example.com",
    "database.port": "1521",
    "database.user": "dbz_user",
    "database.password": "${file:/secrets/sap-password}",
    "database.dbname": "MDGPROD",
    "database.server.name": "sap.mdg",

    "table.include.list": "MDG.MATERIAL,MDG.VENDOR,MDG.CUSTOMER",
    "schema.history.internal.kafka.topic": "sap.mdg.schema_history",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",

    "transforms": "route,unwrap",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "bronze.debezium.$3",

    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite"
  }
}
```

**Output topic structure:**
```
bronze.debezium.material
bronze.debezium.vendor
bronze.debezium.customer
```

---

## Sink Connector Pattern

**Use case:** Write Kafka messages to external system (Iceberg, S3, database)

```json
{
  "name": "iceberg-bronze-sink",
  "config": {
    "connector.class": "io.tabular.iceberg.connect.IcebergSinkConnector",
    "tasks.max": "4",
    "topics": "bronze.debezium.material,bronze.debezium.vendor",

    "iceberg.catalog": "glue",
    "iceberg.catalog.glue.warehouse": "s3://data-lake/warehouse",
    "iceberg.table": "bronze.{topic}",
    "iceberg.table.auto-create": "true",

    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",

    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "dlq.iceberg_sink",
    "errors.deadletterqueue.context.headers.enable": "true"
  }
}
```

---

## Connect Cluster Deployment

```yaml
# docker-compose.yml
services:
  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.5.0
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_GROUP_ID: connect-cluster
      CONNECT_CONFIG_STORAGE_TOPIC: connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: connect-status
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_PLUGIN_PATH: /usr/share/java,/usr/share/confluent-hub-components
    volumes:
      - ./connectors:/usr/share/confluent-hub-components
```

---

## Topic Configuration Details

### Partition Count Formulas

```
Desired Throughput / Consumer Throughput = Min Partitions

Example:
- Target: 100 MB/s throughput
- Per-consumer: 10 MB/s
- Min partitions: 100 / 10 = 10 partitions

Guidelines:
- Start with partitions = max(expected consumers, throughput requirement)
- Over-partition rather than under (easier to add consumers than partitions)
- Typical: 12-24 partitions per topic (2^N for clean distribution)
- Limit: 4,000 partitions per broker (Kafka recommendation)
```

### Retention Policy Configurations

| Policy | When to Use | Configuration |
|--------|-------------|---------------|
| **Time-based** | Log data, events | `retention.ms=604800000` (7 days) |
| **Size-based** | Limit storage cost | `retention.bytes=10737418240` (10 GB) |
| **Compaction** | State changes, CDC | `cleanup.policy=compact` |
| **Infinite** | Event sourcing, audit | `retention.ms=-1` (infinite) |

### Compaction Pattern (for CDC)

```properties
# Retain latest value per key (state compaction)
cleanup.policy=compact
min.cleanable.dirty.ratio=0.5
delete.retention.ms=86400000  # Keep tombstones 1 day
```

---

**Note:** For broader CDC pipeline patterns, see references/integration-patterns.md
