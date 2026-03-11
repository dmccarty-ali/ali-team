# ali-sap-mdg - Licensing and Sizing Reference

This reference contains detailed licensing requirements, platform availability, and sizing guidelines for SAP MDG implementations.

---

## Licensing

### SAP MDG is NOT a Separate Product License

**Key Point:** SAP MDG functionality is included in certain SAP product licenses, not sold separately.

---

### Included In

**1. S/4HANA On-Premise (2021 or later)**
- Full MDG capabilities included in base license
- No additional license cost
- Available in all S/4HANA editions (Enterprise, Professional)
- Applies to both hub and co-deployment models

**2. S/4HANA Cloud Private Edition (RISE with SAP)**
- Full MDG capabilities included
- Managed service (SAP operates infrastructure)
- Hub or co-deployment models available
- Included in RISE subscription pricing

---

### NOT Included In

**1. SAP ECC (Legacy Product)**
- Requires separate SAP MDG 9.x license
- Licensed per-user (named users or concurrent users)
- Typical cost: $5K-$15K per named user
- ECC MDG is end-of-maintenance (2027), migrate to S/4HANA

**2. S/4HANA Cloud Public Edition (Multi-Tenant SaaS)**
- MDG NOT available in public cloud
- Alternative: SAP Master Data Governance on SAP BTP
  - Limited functionality (not full MDG)
  - Separate BTP subscription required
  - Focuses on simple data quality, not consolidation

---

### Licensing Summary Table

| Platform | MDG Included? | License Type | Notes |
|----------|---------------|--------------|-------|
| **S/4HANA On-Prem (2021+)** | ✅ Yes | Base S/4HANA license | Full MDG |
| **S/4HANA Cloud Private** | ✅ Yes | RISE subscription | Full MDG |
| **SAP ECC** | ❌ No | Separate MDG 9.x license | Legacy, EOL 2027 |
| **S/4HANA Cloud Public** | ❌ No | Not available | Use BTP alternative |
| **SAP BTP MDG** | ⚠️ Partial | BTP subscription | Limited functionality |

---

## Platform Availability

### S/4HANA On-Premise

**Deployment Options:**
- Hub deployment (dedicated S/4HANA instance for MDG)
- Co-deployment (MDG in existing transactional S/4HANA)

**Versions:**
- S/4HANA 2021 or later required
- S/4HANA 2023 recommended (latest features)

**Infrastructure:**
- On-premise servers (your data center)
- Private cloud (hosted by third party, e.g., AWS, Azure)

---

### S/4HANA Cloud Private Edition (RISE with SAP)

**Deployment Options:**
- Hub deployment
- Co-deployment

**Managed Service:**
- SAP operates infrastructure
- SAP applies patches and upgrades
- Customer configures MDG (BRFplus, workflows)

**Hosting:**
- SAP-managed data centers
- Or customer-preferred hyperscaler (AWS, Azure, GCP)

**Subscription Model:**
- Monthly or annual subscription
- Includes infrastructure, licenses, and support

---

### SAP BTP Master Data Governance

**Limited Functionality (Not Full MDG):**
- Data quality management
- Simple workflow approval
- REST API for master data CRUD

**NOT Included:**
- Matching and consolidation
- BRFplus workflow engine
- DQM integration
- Multi-domain governance

**Use Case:**
- Small companies (<$100M revenue)
- Simple data quality needs
- Cloud-native architecture (no on-premise systems)

---

## Sizing Considerations

### Factors That Impact Sizing

| Factor | Impact | Recommendation |
|--------|--------|----------------|
| **Number of Records** | Database size, matching performance | >500K records: Hub deployment |
| **Number of Domains** | Complexity, resource usage | 3+ domains: Hub deployment |
| **Number of Users** | Concurrent workflow processing | Budget 2-4 CPU cores per 100 concurrent users |
| **Matching Frequency** | Batch job resource usage | Nightly matching: plan for 2-4 hour batch window |
| **Integration Volume** | Network/API throughput | >10K changes/day: MDI real-time, <10K: IDoc batch |
| **Workflow Complexity** | BRFplus execution time | Complex workflows (>5 steps): +20% CPU |
| **Data Quality (DQM)** | External API calls | Address validation: +10% network bandwidth |

---

## Typical Hub Sizing

### Small Hub (Single Domain, <100K Records)

**Use Case:** Single domain (Customer OR Material), simple workflows, <50 concurrent users

**Infrastructure:**
- **CPU:** 8 cores
- **RAM:** 64 GB
- **Storage:** 500 GB (database + logs)
- **Network:** 1 Gbps

**Estimated Cost:**
- On-premise: $50K-$75K (servers)
- Cloud (RISE): $5K-$8K/month

---

### Medium Hub (2-3 Domains, 100K-500K Records)

**Use Case:** Multiple domains (Customer + Material + Supplier), moderate workflows, 50-100 concurrent users

**Infrastructure:**
- **CPU:** 16 cores
- **RAM:** 128 GB
- **Storage:** 1 TB (database + logs)
- **Network:** 10 Gbps

**Estimated Cost:**
- On-premise: $150K-$200K (servers)
- Cloud (RISE): $15K-$25K/month

---

### Large Hub (5+ Domains, >500K Records)

**Use Case:** Enterprise-wide MDG (all domains), complex matching/consolidation, 100-200 concurrent users

**Infrastructure:**
- **CPU:** 32 cores (or more)
- **RAM:** 256 GB (or more)
- **Storage:** 2-5 TB (database + logs)
- **Network:** 10 Gbps or higher
- **High Availability:** Active/passive cluster

**Estimated Cost:**
- On-premise: $400K-$600K (servers + HA)
- Cloud (RISE): $40K-$70K/month

---

## Sizing by Workload Type

### Workload 1: Concurrent Users

**Formula:** 2-4 CPU cores per 100 concurrent users

| Concurrent Users | CPU Cores | RAM |
|------------------|-----------|-----|
| 25 | 4 | 32 GB |
| 50 | 8 | 64 GB |
| 100 | 16 | 128 GB |
| 200 | 32 | 256 GB |

**Concurrent user = user actively creating/approving change requests**

---

### Workload 2: Batch Matching Jobs

**Formula:** Matching throughput ~10K-50K comparisons/hour per CPU core

| Records to Match | Comparisons (n²) | CPU Cores | Duration |
|------------------|------------------|-----------|----------|
| 10K | 100M | 4 | 2-4 hours |
| 50K | 2.5B | 8 | 8-12 hours |
| 100K | 10B | 16 | 12-16 hours |
| 500K | 250B | 32+ | 16-24 hours |

**Note:** Use blocking strategy (compare within blocks) to reduce comparisons

---

### Workload 3: Integration Volume

**Formula:** 1 Gbps can handle ~10K records/hour (MDI/IDoc)

| Changes/Day | Changes/Hour (peak) | Network Bandwidth |
|-------------|---------------------|-------------------|
| 1K | 100 | 1 Gbps |
| 10K | 1K | 1 Gbps (sufficient) |
| 100K | 10K | 10 Gbps (recommended) |
| 1M | 100K | 10 Gbps or dedicated line |

---

## Storage Sizing

### Database Storage

**Formula:** Estimate 10-50 KB per master record (depends on domain)

| Domain | Size per Record | 100K Records | 500K Records |
|--------|----------------|--------------|--------------|
| Customer (BP) | 20-30 KB | 2-3 GB | 10-15 GB |
| Material | 30-50 KB | 3-5 GB | 15-25 GB |
| Supplier (BP) | 20-30 KB | 2-3 GB | 10-15 GB |

**Add:**
- Change request history (3-5x master data size over time)
- Audit logs (10-20% of data size)
- Indexes (30-50% of data size)

**Example calculation:**
- 500K customers: 15 GB
- Change history (3 years): 45 GB
- Audit logs: 6 GB
- Indexes: 20 GB
- **Total: ~90 GB for Customer domain**

---

### Log Storage

**Formula:** Plan 20-30% of database size for logs

| Database Size | Log Storage |
|---------------|-------------|
| 100 GB | 20-30 GB |
| 500 GB | 100-150 GB |
| 1 TB | 200-300 GB |
| 2 TB | 400-600 GB |

---

## High Availability (HA) Considerations

### Active/Passive Cluster

**Use when:** Mission-critical MDG (can't tolerate downtime)

**Architecture:**
```
Active Node (Primary)
  - Handles all user traffic
  - Runs batch jobs

Passive Node (Standby)
  - Monitors active node health
  - Takes over if active fails (5-10 min failover)

Shared Storage
  - Database files
  - Log files
```

**Sizing:** 2x the cost of single node (two servers, shared storage)

---

### Active/Active Cluster (Advanced)

**Use when:** Very large user base (>500 concurrent users)

**Architecture:**
```
Load Balancer
  ↓
Node 1 (Active) ←→ Node 2 (Active)
  ↓                    ↓
     Shared Database
```

**Sizing:** 2x+ the cost of single node, plus load balancer

---

## Performance Benchmarks

### Expected Performance Metrics

| Metric | Target | Acceptable |
|--------|--------|------------|
| **Change request creation** | <2 seconds | <5 seconds |
| **Workflow approval** | <1 second | <3 seconds |
| **Matching (10K records)** | 2-4 hours | <8 hours |
| **Address validation (DQM)** | <1 second | <3 seconds |
| **OData API response** | <500ms | <2 seconds |
| **IDoc processing** | <5 seconds | <30 seconds |

---

## Cost Estimation

### One-Time Costs (On-Premise)

| Component | Small Hub | Medium Hub | Large Hub |
|-----------|-----------|------------|-----------|
| Hardware | $50K | $150K | $400K |
| SAP Licenses (S/4HANA) | Included | Included | Included |
| Implementation Services | $200K | $500K | $1M+ |
| DQM Provider (Loqate) | $10K/year | $25K/year | $50K/year |
| **Total First Year** | **~$260K** | **~$675K** | **~$1.45M** |

---

### Recurring Costs (RISE with SAP / Cloud)

| Component | Small Hub | Medium Hub | Large Hub |
|-----------|-----------|------------|-----------|
| RISE Subscription | $5K-$8K/month | $15K-$25K/month | $40K-$70K/month |
| DQM Provider | $1K/month | $2K/month | $4K/month |
| **Total Monthly** | **~$6K-$9K** | **~$17K-$27K** | **~$44K-$74K** |
| **Total Annual** | **~$72K-$108K** | **~$204K-$324K** | **~$528K-$888K** |

---

## Rightsizing Recommendations

### When to Upsize

- Change request processing time >5 seconds (add CPU/RAM)
- Matching jobs taking >8 hours (add CPU, implement blocking)
- Concurrent user load >80% (add CPU)
- Database storage >80% full (add storage)
- Integration lag >30 minutes (add network bandwidth)

### When to Downsize

- CPU utilization consistently <20% (reduce cores)
- RAM utilization <40% (reduce memory)
- Storage utilization <50% after 1 year (reduce storage)

### Cost Optimization Tips

1. **Start small, scale up:** Begin with small hub, add resources as needed
2. **Use co-deployment for single domain:** Save infrastructure cost
3. **Batch matching overnight:** Avoid peak-hour resource contention
4. **Archive old change requests:** Reduce database size (archive after 3 years)
5. **Optimize BRFplus rules:** Complex rules consume more CPU
