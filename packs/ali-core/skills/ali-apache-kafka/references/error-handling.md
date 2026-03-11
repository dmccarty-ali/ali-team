# ali-apache-kafka - Error Handling Reference

## Overview

Dead Letter Queue (DLQ) patterns for handling message processing failures without blocking the entire partition.

---

## Producer DLQ Pattern

```java
public void sendWithDLQ(ProducerRecord<String, String> record) {
    try {
        producer.send(record).get();  // Synchronous send
    } catch (Exception e) {
        // Send to DLQ with error context
        ProducerRecord<String, String> dlqRecord = new ProducerRecord<>(
            "dlq." + record.topic(),
            record.key(),
            record.value()
        );

        dlqRecord.headers().add("error.message",
            e.getMessage().getBytes());
        dlqRecord.headers().add("original.topic",
            record.topic().getBytes());
        dlqRecord.headers().add("timestamp",
            String.valueOf(System.currentTimeMillis()).getBytes());

        producer.send(dlqRecord);
    }
}
```

### DLQ Header Fields

Include these headers for debugging and retry logic:

| Header | Purpose | Example |
|--------|---------|---------|
| `error.message` | Exception message | "RecordTooLargeException: Message size 2MB exceeds max 1MB" |
| `error.class` | Exception class name | "org.apache.kafka.common.errors.RecordTooLargeException" |
| `original.topic` | Source topic | "bronze.debezium.material" |
| `original.partition` | Source partition | "5" |
| `original.offset` | Source offset | "48237" |
| `timestamp` | Failure timestamp | "1673019234567" |
| `retry.count` | Number of retries | "3" |
| `source.application` | Application name | "iceberg-sink-v2" |

---

## Consumer DLQ Pattern

```java
ConsumerRecords<String, GenericRecord> records = consumer.poll(Duration.ofMillis(100));

for (ConsumerRecord<String, GenericRecord> record : records) {
    try {
        processRecord(record);
    } catch (Exception e) {
        // Send to DLQ instead of failing entire batch
        sendToDLQ(record, e);
    }
}

consumer.commitSync();  // Commit even if some records failed (DLQ'd)
```

### DLQ Helper Method

```java
private void sendToDLQ(ConsumerRecord<String, GenericRecord> record, Exception e) {
    ProducerRecord<String, GenericRecord> dlqRecord = new ProducerRecord<>(
        "dlq." + record.topic(),
        record.partition(),
        record.key(),
        record.value()
    );

    // Copy original headers
    record.headers().forEach(header ->
        dlqRecord.headers().add(header.key(), header.value())
    );

    // Add error context
    dlqRecord.headers().add("error.message", e.getMessage().getBytes());
    dlqRecord.headers().add("error.class", e.getClass().getName().getBytes());
    dlqRecord.headers().add("original.topic", record.topic().getBytes());
    dlqRecord.headers().add("original.partition",
        String.valueOf(record.partition()).getBytes());
    dlqRecord.headers().add("original.offset",
        String.valueOf(record.offset()).getBytes());
    dlqRecord.headers().add("timestamp",
        String.valueOf(System.currentTimeMillis()).getBytes());

    dlqProducer.send(dlqRecord);
    log.warn("Sent record to DLQ: topic={}, partition={}, offset={}, error={}",
        record.topic(), record.partition(), record.offset(), e.getMessage());
}
```

---

## DLQ Retry Strategy

```java
public void processDLQ() {
    KafkaConsumer<String, String> dlqConsumer = createDLQConsumer();

    while (true) {
        ConsumerRecords<String, String> records =
            dlqConsumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, String> record : records) {
            String errorMsg = new String(
                record.headers().lastHeader("error.message").value()
            );
            String originalTopic = new String(
                record.headers().lastHeader("original.topic").value()
            );

            // Retry logic with exponential backoff
            if (shouldRetry(record)) {
                retryRecord(originalTopic, record);
            } else {
                // Move to permanent failure topic or alert
                logPermanentFailure(record, errorMsg);
            }
        }
    }
}
```

### Retry Decision Logic

```java
private boolean shouldRetry(ConsumerRecord<String, String> record) {
    String errorClass = new String(
        record.headers().lastHeader("error.class").value()
    );

    // Retriable errors
    if (errorClass.contains("TimeoutException") ||
        errorClass.contains("NetworkException") ||
        errorClass.contains("RetriableException")) {

        int retryCount = getRetryCount(record);
        return retryCount < MAX_RETRIES;
    }

    // Non-retriable errors (data quality, schema mismatch)
    return false;
}

private int getRetryCount(ConsumerRecord<String, String> record) {
    Header retryHeader = record.headers().lastHeader("retry.count");
    if (retryHeader == null) {
        return 0;
    }
    return Integer.parseInt(new String(retryHeader.value()));
}
```

### Exponential Backoff

```java
private void retryRecord(String targetTopic, ConsumerRecord<String, String> record) {
    int retryCount = getRetryCount(record);
    long delayMs = (long) Math.pow(2, retryCount) * 1000;  // 1s, 2s, 4s, 8s...

    // Schedule retry after delay
    scheduler.schedule(() -> {
        ProducerRecord<String, String> retryRecord =
            new ProducerRecord<>(targetTopic, record.key(), record.value());

        // Increment retry count
        retryRecord.headers().add("retry.count",
            String.valueOf(retryCount + 1).getBytes());

        producer.send(retryRecord);
    }, delayMs, TimeUnit.MILLISECONDS);
}
```

---

## DLQ Topic Design

### Naming Convention

```
<original-topic>.dlq         # Simple DLQ
dlq.<original-topic>         # Namespaced DLQ
<original-topic>.retry       # Retry queue (before DLQ)
<original-topic>.failed      # Permanent failures
```

**Example:**
```
bronze.debezium.material       # Original topic
bronze.debezium.material.dlq   # Dead letter queue
bronze.debezium.material.failed # Permanent failures (human review)
```

### Multi-Level DLQ Strategy

```
Original Topic → Retry Queue (3 attempts) → DLQ → Failed (manual review)
```

**Flow:**
1. Message fails processing → retry queue (attempt 1-3)
2. Still failing after 3 retries → DLQ (automated investigation)
3. Cannot auto-recover → failed topic (human review)

---

## Kafka Connect DLQ Configuration

Kafka Connect has built-in DLQ support:

```json
{
  "name": "iceberg-bronze-sink",
  "config": {
    "connector.class": "io.tabular.iceberg.connect.IcebergSinkConnector",
    "topics": "bronze.debezium.material",

    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "dlq.iceberg_sink",
    "errors.deadletterqueue.topic.replication.factor": "3",
    "errors.deadletterqueue.context.headers.enable": "true",

    "errors.log.enable": "true",
    "errors.log.include.messages": "true"
  }
}
```

### Connect DLQ Headers

Kafka Connect automatically adds:
- `__connect.errors.topic` - Original topic
- `__connect.errors.partition` - Original partition
- `__connect.errors.offset` - Original offset
- `__connect.errors.connector.name` - Connector name
- `__connect.errors.task.id` - Task ID
- `__connect.errors.stage` - Stage where error occurred
- `__connect.errors.class.name` - Exception class
- `__connect.errors.exception.message` - Error message
- `__connect.errors.exception.stacktrace` - Full stack trace

---

## Monitoring DLQ

### Key Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **DLQ message rate** | Messages/sec going to DLQ | > baseline + 50% |
| **DLQ lag** | Unprocessed DLQ messages | > 1000 |
| **Retry success rate** | % of retries that succeed | < 80% |
| **Permanent failure rate** | Messages moved to .failed | > 10/min |

### Prometheus Metrics

```yaml
- alert: HighDLQRate
  expr: rate(kafka_producer_dlq_messages_total[5m]) > 10
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High rate of messages to DLQ"

- alert: DLQLagHigh
  expr: kafka_consumer_records_lag{topic=~".*\\.dlq"} > 1000
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "DLQ lag exceeds 1000 messages"
```

---

## Error Categories and Handling

| Error Type | Retriable? | Strategy |
|------------|-----------|----------|
| **Network timeout** | Yes | Retry with backoff |
| **Broker unavailable** | Yes | Retry with backoff |
| **Schema mismatch** | No | DLQ → manual review |
| **Data quality** | No | DLQ → data remediation |
| **RecordTooLarge** | No | DLQ → split or reference pattern |
| **Authorization** | No | Alert security team |
| **Deserialization** | No | DLQ → schema fix |
| **Downstream DB unavailable** | Yes | Retry with backoff |
| **Constraint violation** | No | DLQ → data cleansing |

---

## Best Practices

1. **Always use DLQ for production** - Don't block partitions on poison pills
2. **Include rich error context** - Headers with exception, timestamp, retry count
3. **Monitor DLQ lag** - Alert when messages accumulate
4. **Automate retry** - Exponential backoff for retriable errors
5. **Human review for permanent failures** - Send to .failed topic
6. **Set retention on DLQ** - Don't keep forever (7-30 days typical)
7. **Test DLQ flow** - Simulate failures in staging
8. **Alert on DLQ rate spikes** - Indicates systemic issue

---

## Example: Complete Error Handling Flow

```java
public class RobustKafkaConsumer {
    private final KafkaConsumer<String, GenericRecord> consumer;
    private final KafkaProducer<String, GenericRecord> dlqProducer;
    private final KafkaProducer<String, GenericRecord> retryProducer;
    private static final int MAX_RETRIES = 3;

    public void consume() {
        while (true) {
            ConsumerRecords<String, GenericRecord> records =
                consumer.poll(Duration.ofMillis(100));

            for (ConsumerRecord<String, GenericRecord> record : records) {
                try {
                    processRecord(record);
                } catch (RetriableException e) {
                    handleRetriableError(record, e);
                } catch (Exception e) {
                    handleNonRetriableError(record, e);
                }
            }

            consumer.commitSync();
        }
    }

    private void handleRetriableError(ConsumerRecord<String, GenericRecord> record,
                                       Exception e) {
        int retryCount = getRetryCount(record);

        if (retryCount < MAX_RETRIES) {
            sendToRetryQueue(record, e, retryCount + 1);
        } else {
            sendToDLQ(record, e, "Max retries exceeded");
        }
    }

    private void handleNonRetriableError(ConsumerRecord<String, GenericRecord> record,
                                          Exception e) {
        sendToDLQ(record, e, "Non-retriable error");
    }
}
```

---

**Note:** For producer-side error handling, see references/producer-patterns.md
