# ali-apache-kafka - Deployment Options Reference

## Overview

Deployment strategies for Apache Kafka across platforms: self-managed, managed services, and containerized deployments.

---

## Confluent Platform vs Apache Kafka

| Feature | Apache Kafka | Confluent Platform |
|---------|--------------|-------------------|
| **Core broker** | ✓ | ✓ |
| **Schema Registry** | ✗ | ✓ (included) |
| **REST Proxy** | ✗ | ✓ |
| **ksqlDB** | ✓ (separate) | ✓ (included) |
| **Control Center UI** | ✗ | ✓ (commercial) |
| **Confluent Hub** | ✗ | ✓ (connector library) |
| **Replicator** | ✗ | ✓ (cross-cluster) |
| **Tiered Storage** | ✓ (Kafka 3.6+) | ✓ (enhanced) |
| **Support** | Community | Commercial support |
| **License** | Apache 2.0 | Confluent Community License |
| **Cost** | Free | Free (community) / Paid (enterprise) |

### When to Choose Confluent Platform

- Need Schema Registry (highly recommended for production)
- Want Control Center UI for operations
- Need commercial support
- Using Confluent-specific connectors
- Multi-datacenter replication (Replicator)

### When to Choose Apache Kafka

- Pure open-source requirement
- Schema Registry deployed separately
- Custom tooling for operations
- Cost-sensitive environment

---

## Managed Services

### Amazon MSK (Managed Streaming for Kafka)

**Pros:**
- Fully managed (AWS handles broker patching, ZooKeeper)
- Integrated with AWS services (IAM, CloudWatch, VPC)
- Auto-scaling (with MSK Serverless)
- Automatic recovery from broker failures

**Cons:**
- Kafka version lag (typically 1-2 versions behind)
- No Schema Registry included (deploy separately or use Glue Schema Registry)
- Limited configuration options (some broker configs locked)
- AWS-only (no multi-cloud)

**Configuration:**
```bash
# Create MSK cluster via AWS CLI
aws kafka create-cluster \
  --cluster-name my-kafka-cluster \
  --kafka-version 3.4.0 \
  --number-of-broker-nodes 3 \
  --broker-node-group-info \
    InstanceType=kafka.m5.large,ClientSubnets=subnet-abc,subnet-def,subnet-ghi,SecurityGroups=sg-123 \
  --encryption-info \
    EncryptionInTransit={ClientBroker=TLS,InCluster=true}
```

**Monitoring:**
```bash
# CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Kafka \
  --metric-name BytesInPerSec \
  --dimensions Name=Cluster\ Name,Value=my-kafka-cluster
```

---

### Confluent Cloud

**Pros:**
- Full Confluent Platform features
- Multi-cloud (AWS, Azure, GCP)
- Schema Registry included
- ksqlDB fully managed
- Auto-scaling
- Global availability

**Cons:**
- Higher cost than MSK
- Vendor lock-in
- Limited access to broker configs
- No VPC peering (uses public endpoints with auth)

**Pricing model:**
- Pay per throughput (MB/s ingress/egress)
- Pay per storage
- Pay per connector task

**When to choose:**
- Multi-cloud strategy
- Want full Confluent features
- Need ksqlDB or Schema Registry
- Willing to pay premium for convenience

---

### Azure Event Hubs (Kafka-Compatible)

**Pros:**
- Azure-native integration
- Kafka-compatible API
- Auto-scaling
- Lower cost than Confluent Cloud

**Cons:**
- Not true Kafka (different internals)
- Feature limitations (no log compaction, transactions)
- Partition limit (32 per topic)
- Consumer group limit

**Use case:** Azure-only environment, lightweight Kafka needs

---

### Aiven for Apache Kafka

**Pros:**
- Multi-cloud (AWS, Azure, GCP, DigitalOcean)
- Open-source focus (no vendor lock-in)
- Managed Schema Registry, Kafka Connect
- Terraform support
- Transparent pricing

**Cons:**
- Smaller provider (less brand recognition)
- Fewer integrations than Confluent

**Use case:** Multi-cloud, open-source commitment

---

## KRaft Mode (No ZooKeeper)

**Kafka 3.0+ introduces KRaft mode, removing ZooKeeper dependency.**

### Benefits

- Simpler architecture (one less component)
- Faster metadata operations
- Millions of partitions supported (vs thousands with ZK)
- Faster cluster recovery
- Easier to operate

### Configuration

```properties
# server.properties for KRaft
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093

# Metadata log configuration
metadata.log.dir=/var/kafka/metadata

# Listener configuration
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
controller.listener.names=CONTROLLER
```

### Format Metadata Directory

```bash
# Format storage (first time only)
kafka-storage.sh format \
  -t $(kafka-storage.sh random-uuid) \
  -c config/kraft/server.properties
```

### Start KRaft Broker

```bash
kafka-server-start.sh config/kraft/server.properties
```

---

## Migration from ZooKeeper to KRaft

### Kafka 3.x Migration Path

**Current state (Kafka 3.6):**
- No in-place migration from ZK to KRaft
- Must deploy new KRaft cluster
- Use MirrorMaker 2 to replicate data

**Migration steps:**
1. Deploy new Kafka cluster in KRaft mode
2. Configure MirrorMaker 2 (ZK cluster → KRaft cluster)
3. Replicate topics and offsets
4. Cut over producers/consumers to new cluster
5. Decommission ZK cluster

**Future (Kafka 4.0+):**
- ZooKeeper support removed entirely
- All new deployments must use KRaft

---

## Containerized Deployment (Docker)

### Docker Compose Stack

```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_LOG_RETENTION_HOURS: 168
    volumes:
      - kafka-data:/var/lib/kafka/data

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.5.0
    depends_on:
      - kafka
      - schema-registry
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_GROUP_ID: connect-cluster
      CONNECT_CONFIG_STORAGE_TOPIC: connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: connect-status
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
      CONNECT_PLUGIN_PATH: /usr/share/java,/usr/share/confluent-hub-components
    volumes:
      - ./connectors:/usr/share/confluent-hub-components

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
```

---

## Kubernetes Deployment (Strimzi Operator)

### Install Strimzi Operator

```bash
# Add Helm repo
helm repo add strimzi https://strimzi.io/charts/
helm repo update

# Install Strimzi
helm install strimzi-kafka-operator strimzi/strimzi-kafka-operator \
  --namespace kafka \
  --create-namespace
```

### Deploy Kafka Cluster

```yaml
# kafka-cluster.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.5.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
    storage:
      type: persistent-claim
      size: 100Gi
      class: gp3
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      class: gp3
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

```bash
kubectl apply -f kafka-cluster.yaml
```

---

## High Availability Patterns

### Multi-AZ Deployment (AWS)

```
Availability Zone A:
- Broker 1 (leader for partitions 0, 3, 6)
- ZooKeeper 1

Availability Zone B:
- Broker 2 (leader for partitions 1, 4, 7)
- ZooKeeper 2

Availability Zone C:
- Broker 3 (leader for partitions 2, 5, 8)
- ZooKeeper 3

Replication factor: 3
Min ISR: 2
```

**Benefits:**
- Survives AZ failure
- Zero data loss with min.insync.replicas=2
- Automatic failover

**Considerations:**
- Cross-AZ network latency (~1-2ms)
- Cross-AZ data transfer costs

---

### Multi-Region Replication

**Use cases:**
- Disaster recovery
- Global data distribution
- Compliance (data residency)

**Tools:**
- **MirrorMaker 2:** Apache Kafka built-in
- **Confluent Replicator:** Commercial, more features
- **Cluster Linking:** Confluent Cloud

**MirrorMaker 2 example:**
```properties
# mm2.properties
clusters = primary, secondary
primary.bootstrap.servers = kafka-us-east:9092
secondary.bootstrap.servers = kafka-eu-west:9092

primary->secondary.enabled = true
primary->secondary.topics = my-topic.*

replication.factor = 3
sync.topic.configs.enabled = true
```

```bash
connect-mirror-maker.sh mm2.properties
```

---

## Capacity Planning

### Broker Sizing

**Formula:**
```
Brokers needed = (Total throughput / Broker capacity) * Replication factor * Headroom

Example:
- Total throughput: 500 MB/s
- Broker capacity: 100 MB/s
- Replication factor: 3
- Headroom: 1.5x (for peaks)

Brokers = (500 / 100) * 1.5 = 7.5 → 9 brokers
```

### Storage Sizing

```
Storage needed = Daily data * Retention days * Replication factor

Example:
- Daily data: 2 TB/day
- Retention: 7 days
- Replication factor: 3

Storage = 2 TB * 7 * 3 = 42 TB
Provision: 50-60 TB (overhead for compaction, temp files)
```

### Network Bandwidth

```
Network = Producer throughput + (Consumer throughput * Consumers) + Replication

Example:
- Producer: 500 MB/s
- Consumers: 3 groups, 200 MB/s each = 600 MB/s
- Replication: 500 MB/s (same as producer)

Network = 500 + 600 + 500 = 1.6 GB/s
NIC: 10 Gbps minimum (allows headroom)
```

---

## Deployment Best Practices

1. **Use KRaft for new clusters** - Simpler, faster, more scalable
2. **Deploy across AZs** - High availability
3. **Set replication factor = 3** - Standard for production
4. **Use min.insync.replicas = 2** - Balance durability and availability
5. **Separate data and logs** - Use different disks
6. **Monitor from day one** - Prometheus + Grafana
7. **Plan for growth** - Start with headroom (30-50%)
8. **Use managed services** - If expertise is limited
9. **Test disaster recovery** - Practice failover scenarios
10. **Document runbooks** - Operational procedures

---

**Note:** For monitoring cloud deployments, see references/monitoring.md
