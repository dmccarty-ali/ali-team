# ali-apache-kafka - Security Configuration Reference

## Overview

Security patterns for Kafka authentication (SASL), encryption (SSL/TLS), and authorization (ACLs).

---

## SASL/SSL Configuration

### Broker Configuration

```properties
# server.properties
listeners=SASL_SSL://kafka:9093
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN,SCRAM-SHA-512

# SSL settings
ssl.keystore.location=/var/private/ssl/kafka.server.keystore.jks
ssl.keystore.password=${file:/secrets/keystore-password}
ssl.key.password=${file:/secrets/key-password}
ssl.truststore.location=/var/private/ssl/kafka.server.truststore.jks
ssl.truststore.password=${file:/secrets/truststore-password}

# ACLs
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
allow.everyone.if.no.acl.found=false
```

### SASL Mechanisms

| Mechanism | Security | Use Case |
|-----------|----------|----------|
| **PLAIN** | Low (plaintext password) | Development, behind VPN |
| **SCRAM-SHA-256** | Medium | Production with password auth |
| **SCRAM-SHA-512** | High | Production (recommended) |
| **GSSAPI (Kerberos)** | High | Enterprise environments |
| **OAUTHBEARER** | High | OAuth 2.0 integration |

---

## Client Configuration

### Java Producer/Consumer

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9093");
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "SCRAM-SHA-512");
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.scram.ScramLoginModule required " +
    "username=\"producer-user\" " +
    "password=\"${file:/secrets/producer-password}\";");

props.put("ssl.truststore.location", "/var/private/ssl/client.truststore.jks");
props.put("ssl.truststore.password", "${file:/secrets/truststore-password}");
```

### Python Client (kafka-python)

```python
from kafka import KafkaProducer

producer = KafkaProducer(
    bootstrap_servers='kafka:9093',
    security_protocol='SASL_SSL',
    sasl_mechanism='SCRAM-SHA-512',
    sasl_plain_username='producer-user',
    sasl_plain_password='secret-password',
    ssl_cafile='/var/private/ssl/ca-cert.pem',
    ssl_check_hostname=True
)
```

### CLI Configuration

```bash
# client.properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin-secret";

ssl.truststore.location=/var/private/ssl/client.truststore.jks
ssl.truststore.password=truststore-password
```

```bash
# Use with CLI tools
kafka-console-producer \
  --bootstrap-server kafka:9093 \
  --topic my-topic \
  --producer.config client.properties
```

---

## SCRAM User Management

### Create SCRAM User

```bash
# Create user in ZooKeeper (or KRaft)
kafka-configs --bootstrap-server kafka:9092 \
  --alter \
  --add-config 'SCRAM-SHA-512=[password=user-secret]' \
  --entity-type users \
  --entity-name producer-user
```

### List SCRAM Users

```bash
kafka-configs --bootstrap-server kafka:9092 \
  --describe \
  --entity-type users
```

### Delete SCRAM User

```bash
kafka-configs --bootstrap-server kafka:9092 \
  --alter \
  --delete-config 'SCRAM-SHA-512' \
  --entity-type users \
  --entity-name producer-user
```

---

## ACL Management

### Grant Read Access to Consumer Group

```bash
kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:iceberg-consumer \
  --operation Read \
  --topic bronze.debezium.material \
  --group iceberg-bronze-writers
```

### Grant Write Access to Producer

```bash
kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:debezium-producer \
  --operation Write \
  --topic bronze.debezium.*
```

### Grant Admin Access

```bash
# Full cluster admin
kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:admin \
  --operation All \
  --cluster
```

### List ACLs for Topic

```bash
kafka-acls --bootstrap-server kafka:9092 \
  --list \
  --topic bronze.debezium.material
```

### Remove ACL

```bash
kafka-acls --bootstrap-server kafka:9092 \
  --remove \
  --allow-principal User:old-consumer \
  --operation Read \
  --topic bronze.debezium.material
```

---

## ACL Operations

| Operation | Applies To | Description |
|-----------|------------|-------------|
| **Read** | Topic, Group | Consume messages from topic |
| **Write** | Topic | Produce messages to topic |
| **Create** | Topic, Cluster | Create topics |
| **Delete** | Topic | Delete topics |
| **Alter** | Topic, Cluster | Change topic configuration |
| **Describe** | Topic, Group, Cluster | View topic/cluster metadata |
| **ClusterAction** | Cluster | Administrative actions |
| **All** | Any | Grant all permissions |

---

## ACL Patterns

### Producer Pattern

```bash
# Producer needs:
# 1. Write to topic
# 2. Describe topic (metadata)
# 3. Create topic (if auto-create enabled)

kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:my-producer \
  --operation Write \
  --operation Describe \
  --topic my-topic
```

### Consumer Pattern

```bash
# Consumer needs:
# 1. Read from topic
# 2. Describe topic
# 3. Read consumer group offsets

kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:my-consumer \
  --operation Read \
  --operation Describe \
  --topic my-topic

kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:my-consumer \
  --operation Read \
  --group my-consumer-group
```

### Kafka Connect Pattern

```bash
# Connect worker needs:
# 1. Read/Write to connector topics
# 2. Create connector topics (configs, offsets, status)
# 3. Read/Write to user topics

kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:connect-worker \
  --operation Read \
  --operation Write \
  --operation Create \
  --topic connect-*

kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:connect-worker \
  --operation Read \
  --operation Write \
  --topic bronze.debezium.*
```

---

## SSL Certificate Management

### Generate Keystore and Truststore

```bash
# 1. Generate broker keystore
keytool -keystore kafka.server.keystore.jks \
  -alias kafka-broker \
  -genkey \
  -keyalg RSA \
  -keysize 2048 \
  -validity 365

# 2. Generate CA certificate
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365

# 3. Sign broker certificate
keytool -keystore kafka.server.keystore.jks \
  -alias kafka-broker \
  -certreq \
  -file cert-file

openssl x509 -req -CA ca-cert -CAkey ca-key \
  -in cert-file \
  -out cert-signed \
  -days 365 \
  -CAcreateserial

# 4. Import CA and signed cert into keystore
keytool -keystore kafka.server.keystore.jks \
  -alias CARoot \
  -import \
  -file ca-cert

keytool -keystore kafka.server.keystore.jks \
  -alias kafka-broker \
  -import \
  -file cert-signed

# 5. Create truststore with CA cert
keytool -keystore kafka.server.truststore.jks \
  -alias CARoot \
  -import \
  -file ca-cert
```

### Client Truststore

Clients only need truststore (not keystore) for server authentication:

```bash
keytool -keystore client.truststore.jks \
  -alias CARoot \
  -import \
  -file ca-cert
```

---

## Encryption Options

### In-Transit Encryption

| Protocol | Encryption | Authentication | Performance |
|----------|-----------|----------------|-------------|
| **PLAINTEXT** | None | None | Fastest |
| **SSL** | Yes | Optional (2-way) | Moderate overhead |
| **SASL_PLAINTEXT** | None | Yes | Fast |
| **SASL_SSL** | Yes | Yes | Recommended for production |

### At-Rest Encryption

Kafka does not provide built-in at-rest encryption. Options:

1. **Filesystem encryption:** LUKS, dm-crypt (Linux)
2. **Cloud disk encryption:** AWS EBS encryption, Azure Disk Encryption
3. **Application-level encryption:** Encrypt messages before producing

---

## Security Best Practices

1. **Use SASL_SSL in production** - Encrypts traffic and authenticates clients
2. **Disable allow.everyone.if.no.acl.found** - Deny by default
3. **Use SCRAM-SHA-512** - Stronger than PLAIN
4. **Rotate credentials regularly** - 90-day password rotation
5. **Limit super.users** - Only admin accounts
6. **Use ACL patterns** - Wildcard topics for scalability
7. **Enable SSL hostname verification** - Prevent MITM attacks
8. **Store credentials securely** - Use secrets manager, not plaintext files
9. **Audit ACL changes** - Log all ACL modifications
10. **Test ACLs in staging** - Verify before production rollout

---

## Multi-Tenancy Security

### Tenant Isolation via ACLs

```bash
# Tenant A producer
kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:tenant-a-producer \
  --operation Write \
  --topic tenant-a.*

# Tenant A consumer
kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:tenant-a-consumer \
  --operation Read \
  --topic tenant-a.*

# Tenant B (separate users, separate topics)
kafka-acls --bootstrap-server kafka:9092 \
  --add \
  --allow-principal User:tenant-b-producer \
  --operation Write \
  --topic tenant-b.*
```

### Quotas for Resource Isolation

```bash
# Limit tenant throughput
kafka-configs --bootstrap-server kafka:9092 \
  --alter \
  --add-config 'producer_byte_rate=1048576,consumer_byte_rate=2097152' \
  --entity-type users \
  --entity-name tenant-a-producer

# 1 MB/s producer, 2 MB/s consumer
```

---

## Troubleshooting Security

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `SASL authentication failed` | Wrong username/password | Verify SCRAM credentials |
| `Not authorized to access topics` | Missing ACL | Grant appropriate ACL |
| `SSL handshake failed` | Certificate mismatch | Verify truststore contains CA cert |
| `Hostname verification failed` | SSL cert hostname mismatch | Disable verification (dev only) or fix cert |

### Debug SSL

```bash
# Enable SSL debug logging
export KAFKA_OPTS="-Djavax.net.debug=ssl:handshake"

kafka-console-producer \
  --bootstrap-server kafka:9093 \
  --topic test \
  --producer.config client.properties
```

### Test ACLs

```bash
# Describe ACLs for user
kafka-acls --bootstrap-server kafka:9092 \
  --list \
  --principal User:my-user

# Test topic access
kafka-console-producer \
  --bootstrap-server kafka:9093 \
  --topic test-topic \
  --producer.config client.properties
```

---

**Note:** For cloud-specific security (AWS MSK), see references/deployment.md
