# ali-apache-kafka - Consumer Patterns Reference

## Overview

Consumer patterns for reliable, scalable message consumption from Kafka topics.

---

## Consumer Group Pattern

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("group.id", "iceberg-bronze-writers");
props.put("key.deserializer", StringDeserializer.class);
props.put("value.deserializer", KafkaAvroDeserializer.class);
props.put("schema.registry.url", "http://schema-registry:8081");

// Consumer behavior
props.put("enable.auto.commit", "false");      // Manual commit for control
props.put("auto.offset.reset", "earliest");    // Start from beginning if no offset
props.put("max.poll.records", "500");          // Batch size per poll
props.put("max.poll.interval.ms", "300000");   // 5 min processing time allowed

KafkaConsumer<String, GenericRecord> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("bronze.debezium.material"));

try {
    while (true) {
        ConsumerRecords<String, GenericRecord> records =
            consumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, GenericRecord> record : records) {
            processRecord(record);
        }

        consumer.commitSync();  // Commit offsets after processing
    }
} finally {
    consumer.close();
}
```

### Consumer Group Behavior

**Partition assignment:**
- Each partition assigned to ONE consumer in group
- Partitions > consumers: Some consumers get multiple partitions
- Consumers > partitions: Some consumers idle

**Example with 12 partitions:**
- 3 consumers → each gets 4 partitions
- 6 consumers → each gets 2 partitions
- 12 consumers → each gets 1 partition
- 24 consumers → 12 idle (wasted resources)

**Rebalancing triggers:**
- Consumer joins group
- Consumer leaves group (graceful or failure)
- Partition count changes (rare)

---

## Offset Management Strategies

| Strategy | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Auto-commit** | Simple, no code | Risk of data loss | Non-critical logs |
| **commitSync()** | Guaranteed commit | Blocks processing | Exactly-once required |
| **commitAsync()** | Non-blocking | May fail silently | High throughput OK |
| **Manual per-record** | Fine-grained control | Complex code | Critical transactions |

### Auto-Commit (Simplest)

```java
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "5000");  // Every 5 seconds

// Kafka automatically commits offsets in background
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }
}
```

**Risk:** If consumer crashes between auto-commit intervals, some messages reprocessed.

### Manual Sync Commit (Most Reliable)

```java
props.put("enable.auto.commit", "false");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }

    consumer.commitSync();  // Blocks until commit completes
}
```

**Guarantee:** Offsets committed AFTER processing, no message loss.

### Manual Async Commit (Highest Throughput)

```java
props.put("enable.auto.commit", "false");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }

    consumer.commitAsync((offsets, exception) -> {
        if (exception != null) {
            log.error("Commit failed", exception);
        }
    });
}
```

**Benefit:** Non-blocking, high throughput.
**Risk:** Commit may fail silently (callback required).

### Per-Record Commit (Exactly-Once)

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);

        // Commit after each record
        Map<TopicPartition, OffsetAndMetadata> offsets = Map.of(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset() + 1)
        );
        consumer.commitSync(offsets);
    }
}
```

**Use when:** Message processing is expensive, can't afford to reprocess.

---

## Rebalancing Handling

```java
consumer.subscribe(
    Arrays.asList("topic"),
    new ConsumerRebalanceListener() {
        @Override
        public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
            // Save state before losing partitions
            System.out.println("Partitions revoked: " + partitions);
            consumer.commitSync();  // Commit pending work
            saveStateToDatabase();  // Persist any in-memory state
        }

        @Override
        public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
            // Initialize state for new partitions
            System.out.println("Partitions assigned: " + partitions);
            loadStateFromDatabase();  // Load partition-specific state
        }
    }
);
```

### Rebalancing Best Practices

**Before rebalance:**
- Commit offsets (avoid reprocessing)
- Save in-memory state (partition-specific caches)
- Flush pending writes (databases, files)

**After rebalance:**
- Load state for new partitions
- Reset partition-specific metrics
- Verify connectivity to downstream systems

**Minimize rebalance frequency:**
- Increase `max.poll.interval.ms` if processing is slow
- Increase `session.timeout.ms` to tolerate brief network hiccups
- Process batches faster (tune `max.poll.records`)

---

## Consumer Tuning Parameters

| Parameter | Default | Tuned Value | Impact |
|-----------|---------|-------------|--------|
| `fetch.min.bytes` | 1 | 10240-102400 | Wait for more data, reduce requests |
| `fetch.max.wait.ms` | 500 | 100-1000 | Balance latency vs batching |
| `max.poll.records` | 500 | 100-5000 | Batch size per poll (tune to processing time) |
| `max.partition.fetch.bytes` | 1048576 (1 MB) | 2097152-10485760 | Larger messages = larger fetch |
| `max.poll.interval.ms` | 300000 (5 min) | 300000-900000 | Max time between polls before rebalance |
| `session.timeout.ms` | 10000 (10 sec) | 30000-60000 | Heartbeat timeout before consumer marked dead |

### Tuning Strategy

**For low-latency applications:**
- Small `fetch.min.bytes` (1-1024)
- Small `fetch.max.wait.ms` (0-100)
- Small `max.poll.records` (10-100)

**For high-throughput applications:**
- Large `fetch.min.bytes` (102400-1048576)
- Moderate `fetch.max.wait.ms` (100-500)
- Large `max.poll.records` (1000-5000)

**For slow processing:**
- Increase `max.poll.interval.ms` (600000-900000)
- Reduce `max.poll.records` (50-200)
- Increase session timeout (45000-60000)

---

## Partition Assignment Strategies

```java
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.RangeAssignor");  // Default
```

### Available Strategies

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| **RangeAssignor** | Divides partitions by topic | Default, simple |
| **RoundRobinAssignor** | Distributes evenly across consumers | Balanced load |
| **StickyAssignor** | Minimizes partition movement | Reduce rebalance impact |
| **CooperativeStickyAssignor** | Incremental rebalancing | Kafka 2.4+, least disruption |

**Recommendation:** Use CooperativeStickyAssignor for production (Kafka 2.4+)

---

## Seeking to Specific Offsets

### Seek to Beginning

```java
consumer.subscribe(Arrays.asList("topic"));
consumer.poll(Duration.ofMillis(0));  // Trigger partition assignment

consumer.seekToBeginning(consumer.assignment());  // All assigned partitions
```

### Seek to End

```java
consumer.seekToEnd(consumer.assignment());
```

### Seek to Specific Offset

```java
TopicPartition partition = new TopicPartition("topic", 0);
consumer.assign(Arrays.asList(partition));  // Manual assignment
consumer.seek(partition, 12345);  // Specific offset
```

### Seek to Timestamp

```java
Map<TopicPartition, Long> timestampsToSearch = Map.of(
    new TopicPartition("topic", 0), 1609459200000L  // 2021-01-01 00:00:00 UTC
);

Map<TopicPartition, OffsetAndTimestamp> offsets =
    consumer.offsetsForTimes(timestampsToSearch);

offsets.forEach((partition, offsetAndTimestamp) -> {
    if (offsetAndTimestamp != null) {
        consumer.seek(partition, offsetAndTimestamp.offset());
    }
});
```

---

## Graceful Shutdown

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Shutting down consumer...");
    consumer.wakeup();  // Interrupt poll()
}));

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, String> record : records) {
            processRecord(record);
        }

        consumer.commitSync();
    }
} catch (WakeupException e) {
    // Expected during shutdown
    System.out.println("Consumer woken up for shutdown");
} finally {
    consumer.commitSync();  // Final commit
    consumer.close();
    System.out.println("Consumer closed");
}
```

---

## Filtering Messages

### Filter by Header

```java
for (ConsumerRecord<String, String> record : records) {
    Header sourceHeader = record.headers().lastHeader("source-system");

    if (sourceHeader != null && "SAP-MDG".equals(new String(sourceHeader.value()))) {
        processRecord(record);
    }
}
```

### Filter by Key Pattern

```java
for (ConsumerRecord<String, String> record : records) {
    if (record.key() != null && record.key().startsWith("MAT-")) {
        processMaterialRecord(record);
    }
}
```

### Filter by Value Content

```java
for (ConsumerRecord<String, GenericRecord> record : records) {
    GenericRecord value = record.value();
    String type = value.get("type").toString();

    if ("CRITICAL".equals(type)) {
        processCriticalRecord(record);
    }
}
```

---

**Note:** For DLQ consumer patterns, see references/error-handling.md
