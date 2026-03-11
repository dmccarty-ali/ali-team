# ali-apache-kafka - Monitoring Reference

## Overview

Comprehensive monitoring patterns for Kafka clusters, including metrics collection, alerting, and dashboards.

---

## Key Metrics

### Broker Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **UnderReplicatedPartitions** | Partitions without all replicas in sync | > 0 |
| **OfflinePartitionsCount** | Partitions with no leader | > 0 |
| **ActiveControllerCount** | Number of active controllers | != 1 |
| **BytesInPerSec** | Producer throughput | Baseline ±50% |
| **BytesOutPerSec** | Consumer throughput | Baseline ±50% |
| **MessagesInPerSec** | Message rate | Baseline ±50% |
| **RequestsPerSec** | Client request rate | Baseline ±50% |
| **ISRShrinkRate** | Replicas falling out of sync | > 0 frequently |
| **LeaderElectionRate** | Controller failovers | > 1 per hour |
| **NetworkProcessorAvgIdlePercent** | Network thread utilization | < 30% (70%+ busy) |

### Producer Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **record-send-rate** | Records/sec produced | Baseline ±50% |
| **record-error-rate** | Failed sends/sec | > 0 |
| **request-latency-avg** | Average request latency | > 100ms |
| **buffer-available-bytes** | Available send buffer | < 10% total |
| **batch-size-avg** | Average batch size | Monitor for tuning |

### Consumer Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| **records-lag** | Messages behind per partition | > 10,000 |
| **records-lag-max** | Max lag across partitions | > 50,000 |
| **fetch-latency-avg** | Average fetch latency | > 100ms |
| **fetch-rate** | Fetch requests/sec | Baseline ±50% |
| **records-consumed-rate** | Records consumed/sec | Baseline ±50% |

---

## JMX Metrics Export

### Prometheus JMX Exporter Configuration

```yaml
# jmx_exporter.yml
rules:
  # Broker topic metrics
  - pattern: kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec><>Count
    name: kafka_server_broker_topic_metrics_messages_in_total
    type: COUNTER

  - pattern: kafka.server<type=BrokerTopicMetrics, name=BytesInPerSec><>Count
    name: kafka_server_broker_topic_metrics_bytes_in_total
    type: COUNTER

  # Under-replicated partitions (CRITICAL)
  - pattern: kafka.server<type=ReplicaManager, name=UnderReplicatedPartitions><>Value
    name: kafka_server_replica_manager_under_replicated_partitions
    type: GAUGE

  # Offline partitions (CRITICAL)
  - pattern: kafka.controller<type=KafkaController, name=OfflinePartitionsCount><>Value
    name: kafka_controller_offline_partitions_count
    type: GAUGE

  # Active controller
  - pattern: kafka.controller<type=KafkaController, name=ActiveControllerCount><>Value
    name: kafka_controller_active_controller_count
    type: GAUGE

  # Consumer lag
  - pattern: kafka.consumer<type=consumer-fetch-manager-metrics, client-id=(.*), topic=(.*), partition=(.*)><>records-lag
    name: kafka_consumer_records_lag
    labels:
      client_id: $1
      topic: $2
      partition: $3
    type: GAUGE

  # Producer metrics
  - pattern: kafka.producer<type=producer-metrics, client-id=(.*)><>record-send-rate
    name: kafka_producer_record_send_rate
    labels:
      client_id: $1
    type: GAUGE

  # Request metrics
  - pattern: kafka.network<type=RequestMetrics, name=TotalTimeMs, request=(.*)><>Mean
    name: kafka_network_request_metrics_total_time_ms
    labels:
      request: $1
    type: GAUGE
```

### Launch JMX Exporter

```bash
# Download JMX exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.19.0/jmx_prometheus_javaagent-0.19.0.jar

# Add to Kafka broker startup
export KAFKA_OPTS="-javaagent:/opt/jmx_prometheus_javaagent-0.19.0.jar=7071:/opt/jmx_exporter.yml"

# Start Kafka
bin/kafka-server-start.sh config/server.properties
```

### Prometheus Scrape Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kafka-brokers'
    static_configs:
      - targets:
        - 'kafka1:7071'
        - 'kafka2:7071'
        - 'kafka3:7071'
    metrics_path: /metrics
    scrape_interval: 30s

  - job_name: 'kafka-producers'
    static_configs:
      - targets:
        - 'producer1:7072'
        - 'producer2:7072'

  - job_name: 'kafka-consumers'
    static_configs:
      - targets:
        - 'consumer1:7073'
        - 'consumer2:7073'
```

---

## Consumer Lag Monitoring

### CLI Tool

```bash
# kafka-consumer-groups tool
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group iceberg-bronze-writers \
  --describe

# Output:
# GROUP                    TOPIC               PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# iceberg-bronze-writers   bronze.debezium.mat 0          48237           48237           0
# iceberg-bronze-writers   bronze.debezium.mat 1          51203           51450           247
# iceberg-bronze-writers   bronze.debezium.mat 2          49885           49885           0
```

### Prometheus Query for Lag

```promql
# Max lag across all partitions for a consumer group
max by (topic, client_id) (kafka_consumer_records_lag)

# Total lag across all partitions
sum by (topic, client_id) (kafka_consumer_records_lag)

# Lag growth rate (messages/sec falling behind)
rate(kafka_consumer_records_lag[5m]) > 0
```

### Burrow (LinkedIn Lag Monitor)

```yaml
# burrow.toml
[zookeeper]
servers = ["zookeeper:2181"]

[kafka.local]
brokers = ["kafka1:9092", "kafka2:9092", "kafka3:9092"]

[consumer.local]
class-name = "kafka"
servers = ["kafka1:9092"]
start-latest = true

[httpserver.default]
address = ":8000"
```

---

## Alerting Rules (Prometheus)

```yaml
groups:
  - name: kafka-critical
    rules:
      - alert: KafkaOfflinePartitions
        expr: kafka_controller_offline_partitions_count > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka has offline partitions"
          description: "{{ $value }} partitions are offline (no leader)"

      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replica_manager_under_replicated_partitions > 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Under-replicated partitions detected"
          description: "{{ $value }} partitions are under-replicated"

      - alert: KafkaNoActiveController
        expr: kafka_controller_active_controller_count != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No active Kafka controller"
          description: "Active controller count: {{ $value }}"

  - name: kafka-warning
    rules:
      - alert: KafkaConsumerLagHigh
        expr: kafka_consumer_records_lag > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Consumer lag high for {{ $labels.topic }}"
          description: "Lag: {{ $value }} messages"

      - alert: KafkaHighRequestLatency
        expr: kafka_network_request_metrics_total_time_ms{request="Produce"} > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Kafka request latency"
          description: "Produce request latency: {{ $value }}ms"

      - alert: KafkaISRShrinkRate
        expr: rate(kafka_server_replica_manager_isr_shrinks_total[5m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "ISR shrink rate elevated"
          description: "Replicas frequently falling out of sync"

  - name: kafka-info
    rules:
      - alert: KafkaConsumerGroupRebalancing
        expr: changes(kafka_consumer_coordinator_rebalance_latency_total[5m]) > 0
        labels:
          severity: info
        annotations:
          summary: "Consumer group rebalanced"
          description: "Group {{ $labels.group }} rebalanced {{ $value }} times"
```

---

## Grafana Dashboards

### Broker Dashboard

Key panels:
1. **Cluster health:** Active controller, offline partitions, under-replicated partitions
2. **Throughput:** BytesIn/BytesOut per broker
3. **Request latency:** Produce, Fetch, Metadata request latencies
4. **Network:** Network processor idle %, request queue size
5. **Disk:** Log flush rate, log size per topic

### Consumer Dashboard

Key panels:
1. **Lag overview:** Max lag per consumer group
2. **Lag by topic:** Lag heatmap across topics/partitions
3. **Consumption rate:** Records consumed/sec per consumer
4. **Fetch latency:** Average fetch latency
5. **Rebalances:** Rebalance frequency per group

### Producer Dashboard

Key panels:
1. **Send rate:** Records sent/sec per producer
2. **Error rate:** Failed sends/sec
3. **Batch size:** Average batch size
4. **Request latency:** Average produce request latency
5. **Buffer usage:** Available buffer bytes

---

## Log Analysis

### Important Log Patterns

#### Broker Logs

```bash
# Under-replicated partitions
grep "UnderReplicated" /var/log/kafka/server.log

# Leader elections
grep "LeaderAndIsr" /var/log/kafka/server.log

# ISR changes
grep "ISR" /var/log/kafka/server.log

# Out of memory errors
grep -i "OutOfMemory" /var/log/kafka/server.log

# GC pauses
grep "GC pause" /var/log/kafka/gc.log
```

#### Client Logs

```bash
# Producer errors
grep "ERROR" /var/log/producer/producer.log

# Consumer rebalances
grep "Rebalance" /var/log/consumer/consumer.log

# Connection errors
grep "Disconnected" /var/log/*/application.log
```

---

## Health Checks

### Broker Health Check

```bash
#!/bin/bash
# kafka-health-check.sh

BROKER="kafka:9092"

# Check broker is reachable
if ! echo "dump" | nc -w 1 $BROKER > /dev/null 2>&1; then
    echo "CRITICAL: Broker unreachable"
    exit 2
fi

# Check under-replicated partitions
UNDER_REPLICATED=$(kafka-broker-api-versions \
  --bootstrap-server $BROKER | grep -c "UnderReplicated")

if [ "$UNDER_REPLICATED" -gt 0 ]; then
    echo "WARNING: $UNDER_REPLICATED under-replicated partitions"
    exit 1
fi

echo "OK: Broker healthy"
exit 0
```

### Consumer Group Health Check

```bash
#!/bin/bash
# consumer-health-check.sh

BROKER="kafka:9092"
GROUP="my-consumer-group"
MAX_LAG=10000

LAG=$(kafka-consumer-groups --bootstrap-server $BROKER \
  --group $GROUP \
  --describe | awk 'NR>1 {sum+=$6} END {print sum}')

if [ "$LAG" -gt "$MAX_LAG" ]; then
    echo "WARNING: Consumer lag $LAG exceeds $MAX_LAG"
    exit 1
fi

echo "OK: Consumer lag $LAG"
exit 0
```

---

## Distributed Tracing

### OpenTelemetry Instrumentation

```java
// Kafka producer with tracing
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Tracer;

Tracer tracer = GlobalOpenTelemetry.getTracer("kafka-producer");

Span span = tracer.spanBuilder("produce-message")
    .setAttribute("messaging.system", "kafka")
    .setAttribute("messaging.destination", "my-topic")
    .startSpan();

try (Scope scope = span.makeCurrent()) {
    producer.send(record);
} finally {
    span.end();
}
```

---

## Monitoring Best Practices

1. **Monitor all three layers:** Brokers, producers, consumers
2. **Alert on rate of change:** Not just absolute values
3. **Set baseline thresholds:** ±50% from normal
4. **Monitor consumer lag continuously:** Critical metric
5. **Track rebalancing frequency:** Indicates instability
6. **Use distributed tracing:** Correlate across services
7. **Retain metrics long-term:** Capacity planning, trend analysis
8. **Dashboard per role:** Operations, developers, business
9. **Test alerts in staging:** Verify before production
10. **Document runbooks:** Link alerts to remediation steps

---

**Note:** For cloud-specific monitoring (MSK CloudWatch), see references/deployment.md
