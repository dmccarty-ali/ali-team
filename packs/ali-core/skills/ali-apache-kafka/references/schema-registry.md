# ali-apache-kafka - Schema Registry Reference

## Overview

Schema Registry provides a centralized service for managing Avro, Protobuf, and JSON schemas for Kafka messages. It enforces compatibility rules to prevent breaking changes.

---

## Avro Schema Pattern

```json
{
  "type": "record",
  "name": "GoldenRecord",
  "namespace": "com.aliunde.governance",
  "fields": [
    {
      "name": "material_number",
      "type": "string",
      "doc": "Unique material identifier"
    },
    {
      "name": "description",
      "type": ["null", "string"],
      "default": null,
      "doc": "Material description (optional)"
    },
    {
      "name": "base_unit",
      "type": "string",
      "doc": "Base unit of measure"
    },
    {
      "name": "created_timestamp",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      },
      "doc": "Record creation timestamp"
    },
    {
      "name": "metadata",
      "type": {
        "type": "map",
        "values": "string"
      },
      "default": {},
      "doc": "Extensible metadata fields"
    }
  ]
}
```

---

## Schema Evolution Compatibility Modes

| Mode | Forward | Backward | Full | Transitive |
|------|---------|----------|------|------------|
| **BACKWARD** | Consumers read new with old schema | ✓ | ✗ | ✗ | ✗ |
| **BACKWARD_TRANSITIVE** | All previous schemas compatible | ✓ | ✗ | ✗ | ✓ |
| **FORWARD** | Consumers read old with new schema | ✗ | ✓ | ✗ | ✗ |
| **FORWARD_TRANSITIVE** | All future schemas compatible | ✗ | ✓ | ✗ | ✓ |
| **FULL** | Both forward and backward | ✓ | ✓ | ✓ | ✗ |
| **FULL_TRANSITIVE** | All versions compatible | ✓ | ✓ | ✓ | ✓ |
| **NONE** | No compatibility checking | ✗ | ✗ | ✗ | ✗ |

**Recommendation: BACKWARD (default) or FULL_TRANSITIVE for production**

---

## Schema Evolution Rules

```json
// Version 1 (original schema)
{
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "name", "type": "string"}
  ]
}

// Version 2 (backward compatible)
{
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}  // New optional field
  ]
}

// Version 3 (NOT backward compatible - breaks consumers)
{
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "full_name", "type": "string"}  // Removed 'name', added 'full_name'
  ]
}
```

### Evolution Guidelines

**Safe changes (backward compatible):**
- Add optional field with default value
- Remove optional field
- Add new schema variant to union type

**Breaking changes:**
- Remove required field
- Rename field
- Change field type
- Add required field without default

---

## Producer with Schema Registry

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("key.serializer", StringSerializer.class);
props.put("value.serializer", KafkaAvroSerializer.class);
props.put("schema.registry.url", "http://schema-registry:8081");
props.put("auto.register.schemas", "true");
props.put("value.subject.name.strategy",
    "io.confluent.kafka.serializers.subject.TopicRecordNameStrategy");

KafkaProducer<String, GenericRecord> producer = new KafkaProducer<>(props);

Schema schema = new Schema.Parser().parse(new File("golden_record.avsc"));
GenericRecord record = new GenericData.Record(schema);
record.put("material_number", "MAT-12345");
record.put("description", "High-strength steel");
record.put("base_unit", "KG");
record.put("created_timestamp", System.currentTimeMillis());

producer.send(new ProducerRecord<>("sap.mdg.golden_records.v1",
    "MAT-12345", record));
```

---

## Consumer with Schema Registry

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("group.id", "golden-record-consumers");
props.put("key.deserializer", StringDeserializer.class);
props.put("value.deserializer", KafkaAvroDeserializer.class);
props.put("schema.registry.url", "http://schema-registry:8081");
props.put("specific.avro.reader", "false");  // Use GenericRecord

KafkaConsumer<String, GenericRecord> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("sap.mdg.golden_records.v1"));

while (true) {
    ConsumerRecords<String, GenericRecord> records =
        consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, GenericRecord> record : records) {
        GenericRecord value = record.value();
        String materialNum = value.get("material_number").toString();
        String description = value.get("description") != null
            ? value.get("description").toString()
            : "N/A";

        processRecord(materialNum, description);
    }
}
```

---

## Schema Registry REST API

### Register Schema

```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"}]}"}' \
  http://schema-registry:8081/subjects/my-topic-value/versions
```

### Get Schema by ID

```bash
curl http://schema-registry:8081/schemas/ids/1
```

### Get Latest Schema for Subject

```bash
curl http://schema-registry:8081/subjects/my-topic-value/versions/latest
```

### Test Compatibility

```bash
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{...}"}' \
  http://schema-registry:8081/compatibility/subjects/my-topic-value/versions/latest
```

---

**Note:** Schema Registry integrates with Kafka Connect - see references/kafka-connect.md for connector patterns
