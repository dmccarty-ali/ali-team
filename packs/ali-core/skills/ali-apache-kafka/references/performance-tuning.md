# ali-apache-kafka - Performance Tuning Reference

## Overview

Performance tuning parameters for producers, consumers, and brokers to optimize throughput, latency, and resource utilization.

---

## Producer Tuning

### Key Parameters

| Parameter | Default | Tuned Value | Impact |
|-----------|---------|-------------|--------|
| `batch.size` | 16384 (16 KB) | 32768-65536 | Larger batches = higher throughput, higher latency |
| `linger.ms` | 0 | 5-20 | Wait to fill batch, trade latency for throughput |
| `compression.type` | none | snappy, lz4 | Reduce network bandwidth, CPU cost |
| `buffer.memory` | 33554432 (32 MB) | 67108864-134217728 | Prevent blocking on send |
| `acks` | 1 | all (3 replicas) | Durability vs throughput |
| `max.in.flight.requests.per.connection` | 5 | 1-5 | Higher = more throughput, less ordering |

### High-Throughput Configuration

```java
Properties props = new Properties();

// Maximize batching
props.put("batch.size", 65536);        // 64 KB
props.put("linger.ms", 20);            // Wait 20ms for batch

// Compression
props.put("compression.type", "lz4");  // Fast compression

// Buffer
props.put("buffer.memory", 134217728); // 128 MB

// Parallelism
props.put("max.in.flight.requests.per.connection", 5);

// Durability trade-off
props.put("acks", "1");                // Leader only (faster)
```

**Achieves:** 100k+ messages/sec, MB/s throughput
**Trade-off:** Higher latency (20-50ms), potential data loss if acks=1

### Low-Latency Configuration

```java
Properties props = new Properties();

// Minimize batching
props.put("batch.size", 0);            // Send immediately
props.put("linger.ms", 0);             // No wait

// No compression (CPU cost)
props.put("compression.type", "none");

// Durability
props.put("acks", "all");              // Wait for all replicas

// Ordering
props.put("max.in.flight.requests.per.connection", 1);  // Strict ordering
```

**Achieves:** <5ms latency
**Trade-off:** Lower throughput, higher network usage

---

## Consumer Tuning

### Key Parameters

| Parameter | Default | Tuned Value | Impact |
|-----------|---------|-------------|--------|
| `fetch.min.bytes` | 1 | 10240-102400 | Wait for more data, reduce requests |
| `fetch.max.wait.ms` | 500 | 100-1000 | Balance latency vs batching |
| `max.poll.records` | 500 | 100-5000 | Batch size per poll (tune to processing time) |
| `max.partition.fetch.bytes` | 1048576 (1 MB) | 2097152-10485760 | Larger messages = larger fetch |
| `max.poll.interval.ms` | 300000 (5 min) | 300000-900000 | Max time between polls |
| `session.timeout.ms` | 10000 (10 sec) | 30000-60000 | Heartbeat timeout |

### High-Throughput Configuration

```java
Properties props = new Properties();

// Fetch large batches
props.put("fetch.min.bytes", 102400);       // Wait for 100 KB
props.put("fetch.max.wait.ms", 500);        // Max wait 500ms
props.put("max.poll.records", 5000);        // Large poll batch
props.put("max.partition.fetch.bytes", 10485760);  // 10 MB per partition

// Processing time
props.put("max.poll.interval.ms", 600000);  // 10 min processing allowed

// Heartbeat
props.put("session.timeout.ms", 30000);     // 30 sec heartbeat timeout
```

**Achieves:** 100k+ messages/sec consumption
**Trade-off:** Higher memory usage, longer processing windows

### Low-Latency Configuration

```java
Properties props = new Properties();

// Fetch immediately
props.put("fetch.min.bytes", 1);            // Don't wait
props.put("fetch.max.wait.ms", 0);          // Return immediately
props.put("max.poll.records", 100);         // Small batches

// Fast heartbeat
props.put("session.timeout.ms", 10000);     // 10 sec
```

**Achieves:** <10ms poll latency
**Trade-off:** More frequent broker requests, higher CPU

---

## Broker Tuning

### Replication

```properties
# server.properties
num.replica.fetchers=4              # Parallel replication threads
replica.lag.time.max.ms=30000       # Max lag before ISR removal
min.insync.replicas=2               # Min replicas for acks=all
```

**Impact:**
- `num.replica.fetchers`: Higher = faster replication, more CPU
- `replica.lag.time.max.ms`: Lower = faster ISR removal, more rebalances
- `min.insync.replicas`: Higher = more durability, lower availability

### Log Retention

```properties
log.retention.hours=168             # 7 days
log.segment.bytes=1073741824        # 1 GB segments
log.retention.check.interval.ms=300000  # Check every 5 min
```

**Tuning guidance:**
- Larger segments = fewer files, slower compaction
- Smaller segments = more files, faster deletion
- Typical: 512 MB - 2 GB segments

### Network and I/O Threads

```properties
num.network.threads=8               # Network I/O threads
num.io.threads=16                   # Disk I/O threads
socket.send.buffer.bytes=102400     # 100 KB send buffer
socket.receive.buffer.bytes=102400  # 100 KB receive buffer
socket.request.max.bytes=104857600  # 100 MB max request size
```

**Tuning guidance:**
- `num.network.threads`: 1 per broker core (8-16 typical)
- `num.io.threads`: 2x network threads (disk-bound workloads)
- Socket buffers: Match network MTU * RTT

### Background Tasks

```properties
num.recovery.threads.per.data.dir=1  # Threads for log recovery
log.cleaner.threads=2                # Compaction threads
log.cleaner.io.max.bytes.per.second=1048576  # 1 MB/s compaction I/O
```

---

## OS-Level Tuning

### Linux Kernel Parameters

```bash
# /etc/sysctl.conf

# Increase file descriptors
fs.file-max = 100000

# TCP tuning
net.core.somaxconn = 1024
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_fin_timeout = 15

# Socket buffers
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 87380 134217728

# Apply changes
sysctl -p
```

### File Descriptor Limits

```bash
# /etc/security/limits.conf
kafka soft nofile 100000
kafka hard nofile 100000

# Verify
ulimit -n
```

### Disk I/O Scheduler

```bash
# Use deadline scheduler for Kafka data disks
echo deadline > /sys/block/sda/queue/scheduler

# Or noop for SSDs
echo noop > /sys/block/sda/queue/scheduler
```

---

## JVM Tuning

### Heap Size

```bash
# Broker JVM options
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"

# Producer/consumer JVM options
export KAFKA_HEAP_OPTS="-Xms1g -Xmx1g"
```

**Guidelines:**
- Broker: 4-8 GB heap (more isn't always better - OS page cache important)
- Producer: 512 MB - 1 GB
- Consumer: 512 MB - 2 GB (depends on batch size)

### Garbage Collection

```bash
# G1GC (recommended for Kafka)
export KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35"

# GC logging
export KAFKA_GC_LOG_OPTS="-Xlog:gc*:file=/var/log/kafka/gc.log:time,tags:filecount=10,filesize=100M"
```

---

## Monitoring Performance

### Key Metrics to Monitor

| Metric | Description | Target |
|--------|-------------|--------|
| **MessagesInPerSec** | Producer throughput | Baseline |
| **BytesInPerSec** | Network in | < 70% NIC capacity |
| **BytesOutPerSec** | Network out | < 70% NIC capacity |
| **RequestLatencyAvg** | Average request latency | < 50ms |
| **UnderReplicatedPartitions** | Partitions not fully replicated | 0 |
| **OfflinePartitionsCount** | Partitions without leader | 0 |
| **ActiveControllerCount** | Controller availability | 1 |

### JMX Metrics for Performance

```bash
# Producer metrics
kafka.producer:type=producer-metrics,client-id=*:record-send-rate
kafka.producer:type=producer-metrics,client-id=*:record-error-rate
kafka.producer:type=producer-metrics,client-id=*:request-latency-avg

# Consumer metrics
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*:records-lag-max
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*:fetch-latency-avg

# Broker metrics
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce
```

---

## Benchmarking

### Producer Benchmark

```bash
kafka-producer-perf-test \
  --topic perf-test \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput 100000 \
  --producer-props \
    bootstrap.servers=kafka:9092 \
    acks=all \
    batch.size=32768 \
    linger.ms=10 \
    compression.type=lz4
```

### Consumer Benchmark

```bash
kafka-consumer-perf-test \
  --topic perf-test \
  --messages 1000000 \
  --threads 4 \
  --bootstrap-server kafka:9092 \
  --consumer-props \
    group.id=perf-consumer \
    max.poll.records=5000 \
    fetch.min.bytes=102400
```

### End-to-End Latency Test

```bash
kafka-run-class kafka.tools.EndToEndLatency \
  kafka:9092 \
  perf-test \
  10000 \
  all \
  1024
```

---

## Performance Anti-Patterns

| Anti-Pattern | Impact | Fix |
|--------------|--------|-----|
| **Too many small batches** | High overhead | Increase `batch.size`, `linger.ms` |
| **No compression** | High network usage | Use `lz4` or `snappy` |
| **Small heap with large batches** | Frequent GC pauses | Increase heap or reduce batch size |
| **Under-provisioned brokers** | High CPU, slow replication | Add brokers or scale vertically |
| **Too many partitions** | Rebalancing storms | Consolidate topics |
| **Synchronous sends** | Low throughput | Use async with callback |
| **Auto-commit every 1 second** | Frequent offset commits | Increase `auto.commit.interval.ms` |
| **Fetching 1 message at a time** | Poor consumer throughput | Increase `max.poll.records` |

---

## Scaling Strategies

### Horizontal Scaling (Add Brokers)

**When:**
- CPU > 80% consistently
- Network bandwidth saturated
- Disk I/O saturated

**How:**
1. Add brokers to cluster
2. Reassign partitions to new brokers
3. Monitor replication lag during migration

### Vertical Scaling (Bigger Machines)

**When:**
- Few large topics (can't partition further)
- Disk I/O bottleneck (use faster disks)
- Network bottleneck (use faster NICs)

**How:**
1. Add broker with larger instance
2. Migrate partitions to new broker
3. Decommission old broker

### Partition Scaling

**When:**
- Consumer lag increasing
- Want more parallelism

**How:**
1. Increase partition count (cannot decrease!)
2. Add consumers to consumer group
3. Messages redistributed automatically

---

**Note:** For deployment-specific tuning (MSK, Confluent Cloud), see references/deployment.md
