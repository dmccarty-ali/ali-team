# ali-starburst - Deployment and Sizing

Deployment options, cluster sizing, and scaling strategies.

---

## Starburst Galaxy (SaaS)

### Characteristics

- Fully managed, AWS/Azure hosting
- Auto-scaling workers
- Built-in monitoring and alerting
- Pay-per-use pricing
- Separation of compute and storage

### Use Cases

- Teams without Kubernetes/infrastructure expertise
- Variable workloads (scale to zero when idle)
- Rapid deployment (minutes)

### Configuration

```yaml
# galaxy.yaml (declarative cluster config)
cluster:
  name: production
  cloud: aws
  region: us-east-1
  size: small  # small, medium, large, xlarge
  autoscaling:
    min_workers: 2
    max_workers: 20

catalogs:
  - name: iceberg
    type: iceberg
    properties:
      iceberg.catalog.type: rest
      iceberg.rest-catalog.uri: https://catalog.example.com
```

### Cluster Sizes

| Size | Coordinator | Worker | Use Case |
|------|------------|--------|----------|
| **Small** | 4 vCPU, 16 GB | 4 vCPU, 16 GB | Development, testing |
| **Medium** | 8 vCPU, 32 GB | 8 vCPU, 32 GB | Small production workloads |
| **Large** | 16 vCPU, 64 GB | 16 vCPU, 64 GB | Production analytics |
| **XLarge** | 32 vCPU, 128 GB | 32 vCPU, 128 GB | Heavy workloads |

---

## Starburst Enterprise (Self-Managed)

### Characteristics

- Kubernetes deployment (EKS, AKS, GKE, on-prem)
- Full control over infrastructure
- Private network deployment
- Bring-your-own license

### Use Cases

- Strict data residency requirements
- On-premises infrastructure
- Custom networking (private VPC, VPN)

### Helm Chart Deployment

```bash
# Add Starburst Helm repo
helm repo add starburst https://harbor.starburstdata.net/chartrepo/starburstdata

# Deploy coordinator and workers
helm install starburst starburst/starburst-enterprise \
  --set coordinator.replicas=1 \
  --set worker.replicas=5 \
  --set worker.autoscaling.enabled=true \
  --set worker.autoscaling.minReplicas=2 \
  --set worker.autoscaling.maxReplicas=20 \
  --set resources.coordinator.memory=32Gi \
  --set resources.worker.memory=64Gi
```

### Kubernetes Resource Definitions

```yaml
# coordinator-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: starburst-coordinator
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: coordinator
        image: starburst/starburst-enterprise:latest
        resources:
          requests:
            memory: "32Gi"
            cpu: "8"
          limits:
            memory: "32Gi"
            cpu: "8"
        env:
        - name: STARBURST_NODE_TYPE
          value: "coordinator"
```

### Storage Configuration

```yaml
# Persistent volume for Warp Speed cache
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: warp-speed-cache
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Ti
  storageClassName: fast-ssd
```

---

## Open-Source Trino

### Characteristics

- Free, community-supported
- Missing enterprise features (Warp Speed, Ranger, data products, Galaxy)
- Manual scaling and monitoring

### Use Cases

- Development environments
- Cost-sensitive workloads
- No governance requirements

### Deployment

```bash
# Download Trino
wget https://repo1.maven.org/maven2/io/trino/trino-server/VERSION/trino-server-VERSION.tar.gz
tar -xzf trino-server-VERSION.tar.gz

# Configure coordinator
cat > etc/config.properties <<EOF
coordinator=true
node-scheduler.include-coordinator=false
http-server.http.port=8080
discovery.uri=http://localhost:8080
EOF

# Start coordinator
bin/launcher start
```

---

## Cluster Sizing

### Coordinator Sizing

| Workload | vCPU | Memory | Notes |
|----------|------|--------|-------|
| **Small** (< 20 users) | 8 | 32 GB | Metadata, planning only |
| **Medium** (20-100 users) | 16 | 64 GB | More concurrent queries |
| **Large** (100+ users) | 32 | 128 GB | High concurrency |

**Coordinator should NOT process data** - use `node-scheduler.include-coordinator=false`

### Worker Sizing

| Workload Type | vCPU per Worker | Memory per Worker | Worker Count |
|---------------|----------------|-------------------|--------------|
| **Ad-hoc analytics** | 8 | 32 GB | Auto-scale 2-20 |
| **BI dashboards** | 16 | 64 GB | Auto-scale 5-50 |
| **ETL pipelines** | 32 | 128 GB | Fixed 10-20 |

### Sizing Rules of Thumb

- **Memory-heavy** (large joins, aggregations): 8 GB memory per vCPU
- **CPU-heavy** (transformations, filters): 4 GB memory per vCPU
- **Minimum worker memory**: 16 GB (smaller causes OOM on joins)
- **Maximum worker memory**: 512 GB (diminishing returns above)

---

## Auto-Scaling Strategy

### Kubernetes HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: starburst-worker
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: starburst-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
    scaleUp:
      stabilizationWindowSeconds: 60   # Scale up quickly
```

### Custom Metrics Scaling

```yaml
# Scale based on queue length
- type: Pods
  pods:
    metric:
      name: starburst_queued_queries
    target:
      type: AverageValue
      averageValue: "5"
```

### Scaling Best Practices

1. **Scale up aggressively**: Users notice slow queries immediately
2. **Scale down conservatively**: Avoid thrashing (up/down/up)
3. **Set min replicas > 0**: Avoid cold start latency
4. **Monitor queue time**: If > 1s, scale up faster
5. **Use node affinity**: Pin workers to specific node pools (cost optimization)

---

## Network Configuration

### Private Networking

```yaml
# Service with internal load balancer
apiVersion: v1
kind: Service
metadata:
  name: starburst-coordinator
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: starburst-coordinator
```

### VPN Integration

```properties
# config.properties
http-server.https.enabled=true
http-server.https.port=8443
http-server.https.keystore.path=/etc/certs/keystore.jks
http-server.https.keystore.key=<password>
```

---

**Last Updated:** 2026-02-16
