---
name: ali-apache-kafka
description: |
  Apache Kafka distributed streaming platform patterns. Use when:

  PLANNING: Designing event-driven architectures, planning topic schemas,
  architecting CDC pipelines, considering Kafka vs alternatives (RabbitMQ,
  SQS, Pulsar), evaluating Kafka Connect vs custom consumers

  IMPLEMENTATION: Building Kafka producers/consumers, implementing Kafka
  Connect connectors, writing Kafka Streams applications, configuring
  schema registry, setting up dead letter queues, implementing exactly-once semantics

  GUIDANCE: Asking about partition key selection, topic retention policies,
  consumer group rebalancing, schema evolution strategies, performance
  tuning, security patterns, monitoring approaches

  REVIEW: Reviewing topic design, checking producer/consumer configurations,
  validating schema compatibility, auditing security setup, checking
  performance tuning parameters, reviewing DLQ error handling
---

# Apache Kafka

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing event-driven architectures with Kafka
- Planning topic schemas and partition strategies
- Architecting CDC pipelines with Debezium
- Evaluating Kafka vs alternatives (RabbitMQ, SQS, Pulsar, Event Hubs)
- Designing stream processing applications

**Implementation:**
- Building Kafka producers and consumers
- Implementing Kafka Connect source/sink connectors
- Writing Kafka Streams or ksqlDB applications
- Configuring Schema Registry with Avro/Protobuf
- Setting up dead letter queues and error handling
- Implementing security (SASL/SSL, ACLs)

**Guidance/Best Practices:**
- Asking about partition key selection
- Asking about topic retention and compaction policies
- Asking about consumer group rebalancing strategies
- Asking about schema evolution compatibility modes
- Asking about performance tuning (batching, compression, fetch sizes)
- Asking about monitoring metrics and alerting

**Review/Validation:**
- Reviewing topic design and naming conventions
- Checking producer/consumer configurations
- Validating schema compatibility settings
- Auditing security configurations
- Reviewing performance tuning parameters
- Checking dead letter queue error handling

---

## Do NOT Use For

- **RabbitMQ implementation** (use ali-rabbitmq skill instead)
- **AWS SQS/SNS** (use ali-aws skill instead)
- **Azure Event Hubs** (use ali-azure skill instead)
- **Apache Pulsar** (separate messaging system, not covered)
- **General message queue patterns** (use ali-event-driven-architecture if exists)
- **Stream processing frameworks** (Kafka Streams/ksqlDB are included, but Flink/Spark Streaming are separate)

---

## Key Principles

- **Topics are the unit of parallelism**: Partitions enable horizontal scaling
- **Partition keys determine ordering**: Same key = same partition = ordering guarantee
- **Consumer groups enable parallel processing**: Each partition assigned to one consumer in group
- **Immutable log semantics**: Messages are append-only, never modified
- **At-least-once by default**: Exactly-once requires idempotent producer + transactional writes
- **Schema evolution is critical**: Backward/forward compatibility prevents breaking consumers
- **Replication provides durability**: min.insync.replicas prevents data loss
- **Dead letter queues isolate failures**: Don't block processing on poison pills

---

## Core Architecture

### Cluster Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Kafka Cluster                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  Broker 1    │  │  Broker 2    │  │  Broker 3    │       │
│  │  (Leader)    │  │  (Follower)  │  │  (Follower)  │       │
│  │              │  │              │  │              │       │
│  │  Partition 0 │  │  Partition 0 │  │  Partition 0 │       │
│  │  Partition 1 │  │  Partition 1 │  │  Partition 1 │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           ZooKeeper / KRaft (metadata)                │   │
│  │  - Broker membership                                  │   │
│  │  - Topic configuration                                │   │
│  │  - Controller election                                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         ▲                           │
         │ Produce                   │ Consume
         │                           ▼
┌─────────────────┐         ┌─────────────────┐
│   Producers     │         │  Consumer Group │
│  - Application  │         │  - Consumer 1   │
│  - CDC (Debez.) │         │  - Consumer 2   │
│  - Connect Src  │         │  - Consumer 3   │
└─────────────────┘         └─────────────────┘
```

### Key Concepts

| Concept | Definition | Example |
|---------|------------|---------|
| **Topic** | Logical stream of events | `collibra.edge.assets`, `sap.mdg.golden_records` |
| **Partition** | Ordered, immutable log within topic | Topic with 12 partitions (0-11) |
| **Offset** | Sequential ID for message in partition | Message at offset 48237 |
| **Producer** | Publishes messages to topics | Debezium CDC connector |
| **Consumer** | Reads messages from topics | Iceberg sink connector |
| **Consumer Group** | Set of consumers sharing load | `iceberg-bronze-writers` group |
| **Broker** | Kafka server node | kafka-broker-1.example.com |
| **Replication** | Duplicate partitions across brokers | Replication factor = 3 |
| **ISR** | In-Sync Replicas (caught up followers) | 3 replicas, 2 in ISR (1 lagging) |

---

## Topic Design

### Naming Conventions

**Pattern:** `<domain>.<system>.<entity>.<version>`

```
# Good naming
collibra.edge.assets.v1
sap.mdg.customers.v1
bronze.debezium.sap_users
silver.curated.golden_records
gold.analytics.customer_360

# Avoid
data                    # Too generic
CollibralEdge          # CamelCase (prefer snake_case or kebab-case)
collibra_edge_assets_v1_final_new  # Too long, unclear versioning
```

### Partition Strategy

| Partition Key | Use Case | Pros | Cons |
|---------------|----------|------|------|
| **Entity ID** | User events, customer records | Guaranteed ordering per entity | Hotspot risk if some IDs very active |
| **Hash of ID** | Evenly distributed load | Balanced partitions | No ordering guarantee across entities |
| **Timestamp** | Time-series data | Good for batch windows | Time-based skew (peak hours) |
| **Null (round-robin)** | No ordering needed | Maximum parallelism | Zero ordering guarantees |
| **Region/Tenant** | Multi-tenant isolation | Tenant-specific scaling | Tenant size imbalance |

**Key takeaways:**
- Over-partition rather than under-partition (easier to add consumers than partitions)
- Typical range: 12-24 partitions per topic
- Use compaction for CDC/state changes, time-based retention for logs

---

## Using This Skill

**Core patterns are in this file. Detailed implementations are in references/.**

When you need implementation details, I will prompt you to read the appropriate reference file:

| Topic | Reference File |
|-------|----------------|
| Kafka Connect connectors (Debezium, sinks) | references/kafka-connect.md |
| Schema Registry (Avro, compatibility, evolution) | references/schema-registry.md |
| Producer patterns (idempotence, performance, partitioners) | references/producer-patterns.md |
| Consumer patterns (groups, offsets, rebalancing) | references/consumer-patterns.md |
| Error handling and DLQs | references/error-handling.md |
| Security (SASL/SSL, ACLs) | references/security-config.md |
| Performance tuning (producer, consumer, broker) | references/performance-tuning.md |
| Monitoring (metrics, alerting, dashboards) | references/monitoring.md |
| Deployment (Confluent, MSK, KRaft, Kubernetes) | references/deployment.md |
| Architecture patterns (Event-Driven, CDC, CQRS, Saga) | references/integration-patterns.md |
| Docker Compose for local development | references/docker-compose.md |

---

## Common Pitfalls

| Pitfall | Why It's Bad | Solution |
|---------|--------------|----------|
| **Too few partitions** | Limits parallelism, can't scale | Start with 12-24, plan for growth |
| **Too many partitions** | Broker overhead, rebalancing storms | Consolidate low-traffic topics |
| **No partition key** | Round-robin, no ordering | Use entity ID as key |
| **Wrong partition key** | Hotspots (e.g., all null keys) | Balance cardinality and distribution |
| **Auto-commit enabled** | Data loss on rebalance | Disable, use manual commit |
| **No DLQ** | Poison pill blocks entire partition | Implement DLQ pattern |
| **acks=1** | Data loss if leader fails | Use acks=all for durability |
| **No schema registry** | Breaking changes, deserialization errors | Use Schema Registry with compatibility |
| **Infinite retention** | Disk fills up | Set retention.ms or retention.bytes |
| **Large messages** | Broker overhead, OOM risk | Use reference pattern (store in S3, send key) |
| **Consumer group rebalancing storms** | Processing stops frequently | Tune max.poll.interval.ms, process faster |
| **No monitoring** | Invisible failures | Monitor lag, under-replicated partitions, ISR |

---

## Decision Guides

### When to Use Kafka vs Alternatives

| Use Case | Kafka | RabbitMQ | SQS | Pulsar | Event Hubs |
|----------|-------|----------|-----|--------|------------|
| **High throughput** | ✓ | ✗ | ✗ | ✓ | ✓ |
| **Event replay** | ✓ | ✗ | ✗ | ✓ | ✓ |
| **Ordering guarantees** | ✓ (partition) | ✓ (queue) | ✓ (FIFO) | ✓ | ✓ |
| **Complex routing** | ✗ | ✓ | ✗ | ✗ | ✗ |
| **Low latency** | ✓ | ✓ | ✗ | ✓ | ✓ |
| **Managed service** | MSK | AmazonMQ | ✓ | ✗ | ✓ |
| **Multi-tenancy** | Manual | ✓ | ✓ | ✓ | ✓ |
| **Geo-replication** | ✓ | ✗ | ✗ | ✓ | ✓ |

**Choose Kafka when:**
- High throughput (millions of messages/sec)
- Event replay required (data lake, analytics)
- CDC pipelines (Debezium integration)
- Stream processing (Kafka Streams, ksqlDB)
- Long retention (days, weeks, infinite)

**Choose RabbitMQ when:**
- Complex routing (topic exchanges, headers)
- Traditional message queue semantics
- Low message volume (<10k/sec)
- Strong delivery guarantees per message

**Choose SQS when:**
- AWS-native architecture
- Simple queue pattern
- No event replay needed
- Managed service priority

**Choose Pulsar when:**
- Multi-tenancy built-in
- Geo-replication critical
- Unified streaming + queuing
- Newer technology acceptable

---

## Quick Reference

### CLI Commands

```bash
# Create topic
kafka-topics --bootstrap-server kafka:9092 \
  --create \
  --topic my-topic \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000

# Describe topic
kafka-topics --bootstrap-server kafka:9092 \
  --describe \
  --topic my-topic

# Produce message
echo "key:value" | kafka-console-producer \
  --bootstrap-server kafka:9092 \
  --topic my-topic \
  --property "parse.key=true" \
  --property "key.separator=:"

# Consume messages
kafka-console-consumer \
  --bootstrap-server kafka:9092 \
  --topic my-topic \
  --from-beginning \
  --property print.key=true

# List consumer groups
kafka-consumer-groups --bootstrap-server kafka:9092 --list

# Check consumer lag
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group my-group \
  --describe

# Reset consumer offsets
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group my-group \
  --topic my-topic \
  --reset-offsets \
  --to-earliest \
  --execute
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Crown governance architecture | Crown Equipment RFP design docs (Collibra Edge → Kafka → Iceberg) |
| Debezium CDC patterns | SAP MDG → Kafka integration |
| Medallion architecture | Bronze layer ingestion via Kafka |
| Schema evolution | Schema Registry integration with Avro |

---

## References

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Confluent Kafka Guide](https://docs.confluent.io/)
- [Kafka: The Definitive Guide (Book)](https://www.confluent.io/resources/kafka-the-definitive-guide/)
- [Debezium CDC](https://debezium.io/)
- [Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Kafka Streams](https://kafka.apache.org/documentation/streams/)
- [ksqlDB](https://ksqldb.io/)

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
