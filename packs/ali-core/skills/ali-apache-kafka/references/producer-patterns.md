# ali-apache-kafka - Producer Patterns Reference

## Overview

Producer patterns for reliable, performant message publishing to Kafka topics.

---

## Idempotent Producer (Exactly-Once)

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("key.serializer", StringSerializer.class);
props.put("value.serializer", StringSerializer.class);

// Exactly-once producer settings
props.put("enable.idempotence", "true");  // Enables idempotence
props.put("acks", "all");                 // Wait for all in-sync replicas
props.put("retries", Integer.MAX_VALUE);  // Retry indefinitely
props.put("max.in.flight.requests.per.connection", "5");  // Kafka 2.0+

// Optional: transactional writes
props.put("transactional.id", "producer-1");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// Initialize transactions (if using transactional.id)
producer.initTransactions();

try {
    producer.beginTransaction();

    producer.send(new ProducerRecord<>("topic", "key1", "value1"));
    producer.send(new ProducerRecord<>("topic", "key2", "value2"));

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

### Idempotence Guarantees

**Without idempotence:**
- Network failures can cause duplicate messages
- Producer retry may resend same message
- Consumer sees duplicates

**With idempotence:**
- Producer assigns sequence number to each message
- Broker deduplicates based on producer ID + sequence
- Exactly-once delivery to partition

**Requirements:**
- `enable.idempotence=true`
- `acks=all` (or -1)
- `retries > 0`
- `max.in.flight.requests.per.connection <= 5`

---

## Performance Tuning

```java
Properties props = new Properties();

// Batching (trade latency for throughput)
props.put("batch.size", 32768);           // 32 KB batch size
props.put("linger.ms", 10);               // Wait 10ms to fill batch

// Compression (reduce network bandwidth)
props.put("compression.type", "snappy");  // snappy, gzip, lz4, zstd

// Buffer memory
props.put("buffer.memory", 67108864);     // 64 MB buffer

// Request timeout
props.put("request.timeout.ms", 30000);   // 30 seconds

// Partitioner
props.put("partitioner.class",
    "com.aliunde.kafka.MaterialNumberPartitioner");  // Custom partitioner
```

### Performance Parameters Explained

| Parameter | Default | Tuned Value | Impact |
|-----------|---------|-------------|--------|
| `batch.size` | 16384 (16 KB) | 32768-65536 | Larger batches = higher throughput, higher latency |
| `linger.ms` | 0 | 5-20 | Wait to fill batch, trade latency for throughput |
| `compression.type` | none | snappy, lz4 | Reduce network bandwidth, CPU cost |
| `buffer.memory` | 33554432 (32 MB) | 67108864-134217728 | Prevent blocking on send |
| `acks` | 1 | all (3 replicas) | Durability vs throughput |

### Compression Type Selection

| Type | Compression Ratio | CPU Cost | Use Case |
|------|------------------|----------|----------|
| **none** | 1.0x | None | Low CPU, fast network |
| **snappy** | 2-3x | Low | Balanced (default choice) |
| **lz4** | 2-3x | Very low | Highest throughput |
| **gzip** | 4-5x | High | Slow network, high CPU |
| **zstd** | 3-4x | Medium | Kafka 2.1+, best compression |

---

## Custom Partitioner

```java
public class MaterialNumberPartitioner extends DefaultPartitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        // Extract material type prefix (MAT-, VEN-, CUST-)
        String materialKey = (String) key;
        String prefix = materialKey.substring(0, 4);

        // Partition by material type to avoid cross-type hotspots
        int partitionCount = cluster.partitionCountForTopic(topic);
        return Math.abs(prefix.hashCode()) % partitionCount;
    }
}
```

### When to Use Custom Partitioners

**Default partitioner behavior:**
- If key present: hash(key) % partition_count
- If key null: round-robin across partitions

**Use custom partitioner when:**
- Need domain-specific partitioning logic
- Want to avoid hotspots (e.g., highly active keys)
- Need to co-locate related messages
- Want to balance by entity type or tenant

**Example use cases:**
- Multi-tenant: partition by tenant ID to isolate workloads
- Time-series: partition by time bucket for batch processing
- Geographic: partition by region for data locality

---

## Asynchronous Send with Callback

```java
producer.send(
    new ProducerRecord<>("topic", "key", "value"),
    new Callback() {
        @Override
        public void onCompletion(RecordMetadata metadata, Exception exception) {
            if (exception != null) {
                // Handle send failure
                log.error("Failed to send message", exception);
                sendToDLQ(record, exception);
            } else {
                // Success
                log.info("Sent to partition {} at offset {}",
                    metadata.partition(), metadata.offset());
            }
        }
    }
);
```

### Synchronous vs Asynchronous

| Approach | Latency | Throughput | Error Handling |
|----------|---------|------------|----------------|
| **Sync** (`get()`) | High | Low | Immediate, blocks on error |
| **Async** (callback) | Low | High | Deferred, non-blocking |
| **Fire-and-forget** | Lowest | Highest | No error handling |

**Recommendation:** Use async with callback for production (balance throughput + reliability)

---

## Producer Error Handling

### Retriable Errors

Kafka automatically retries these errors:
- `NOT_LEADER_FOR_PARTITION` - Broker election in progress
- `REQUEST_TIMEOUT` - Slow network or overloaded broker
- `NOT_ENOUGH_REPLICAS` - Not enough in-sync replicas

**Configuration:**
```java
props.put("retries", Integer.MAX_VALUE);  // Retry indefinitely
props.put("retry.backoff.ms", 100);       // Wait 100ms between retries
props.put("delivery.timeout.ms", 120000); // Total timeout: 2 minutes
```

### Non-Retriable Errors

These require application-level handling:
- `INVALID_TOPIC_EXCEPTION` - Topic doesn't exist
- `RECORD_TOO_LARGE` - Message exceeds max.message.bytes
- `AUTHORIZATION_FAILED` - ACL denial

**Pattern:**
```java
try {
    producer.send(record).get();
} catch (ExecutionException e) {
    if (e.getCause() instanceof RecordTooLargeException) {
        // Split large message or use reference pattern
        sendLargeMessageReference(record);
    } else if (e.getCause() instanceof AuthorizationException) {
        // Alert operations
        alertSecurityTeam(record);
    }
}
```

---

## Header Management

```java
ProducerRecord<String, String> record =
    new ProducerRecord<>("topic", "key", "value");

// Add headers for tracing, routing, metadata
record.headers().add("trace-id", UUID.randomUUID().toString().getBytes());
record.headers().add("source-system", "SAP-MDG".getBytes());
record.headers().add("schema-version", "1.0".getBytes());

producer.send(record);
```

### Common Header Use Cases

- **Tracing:** Correlation IDs for distributed tracing
- **Routing:** Metadata for consumer filtering
- **Schema versioning:** Explicit version without schema registry
- **Security:** Authentication tokens (not recommended - use SSL)
- **Data lineage:** Source system, transformation pipeline

---

**Note:** For DLQ producer patterns, see references/error-handling.md
