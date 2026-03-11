---
name: ali-starburst
description: |
  Starburst and Trino query federation architecture. Use when:

  PLANNING: Designing query federation layers, architecting data virtualization,
  planning connector configurations, considering Starburst vs other approaches,
  designing access control policies, planning Iceberg integration

  IMPLEMENTATION: Configuring Starburst/Trino connectors, implementing
  cross-catalog queries, setting up security (LDAP, Ranger, OPA), creating
  materialized views, implementing data products, configuring Warp Speed cache

  GUIDANCE: Asking about connector pushdown optimization, query federation
  best practices, when to use federation vs ETL, Starburst Galaxy vs Enterprise,
  Iceberg integration patterns, Collibra lineage integration

  REVIEW: Reviewing query federation performance, auditing access control policies,
  validating connector configurations, checking pushdown effectiveness, reviewing
  cluster sizing and auto-scaling

  # Do NOT use for:
  # - Iceberg table format internals (use ali-apache-iceberg skill)
  # - Snowflake-specific features not involving federation (use ali-snowflake-core skill)
  # - General data architecture decisions (use ali-data-architecture skill)
  # - Kafka streaming ingestion (use ali-apache-kafka skill)
  # - Collibra governance workflows (use ali-collibra skill)
---

# Starburst / Trino

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing query federation architecture
- Planning Starburst/Trino deployment (Galaxy SaaS vs Enterprise self-managed)
- Architecting data virtualization layers
- Evaluating Starburst vs alternatives (Snowflake, Databricks, Athena)
- Designing access control and governance integration
- Planning Iceberg REST catalog integration

**Implementation:**
- Configuring connectors (Iceberg, Hive, Delta Lake, RDBMS, S3, Kafka, etc.)
- Building cross-catalog query patterns
- Implementing row-level security (RLS) and column masking
- Setting up authentication (LDAP, AD, OAuth)
- Creating data products and domains
- Configuring Warp Speed caching

**Guidance/Best Practices:**
- Asking about query optimization and pushdown
- Asking when federation adds value vs ETL
- Asking about connector-specific limitations
- Asking about performance tuning
- Asking about Collibra integration for lineage and governance

**Review/Validation:**
- Reviewing query performance and execution plans
- Auditing access control policies
- Validating connector configurations
- Checking cluster resource utilization
- Reviewing slow query patterns

---

## Key Principles

- **Query federation is virtual**: No data movement - queries executed at source when possible
- **Pushdown is critical**: Performance depends on pushing filters/aggregations to source systems
- **Connectors vary widely**: Each connector has different pushdown capabilities and limitations
- **Security is layered**: Authentication (who you are) + Authorization (what you can access) + Data masking
- **Iceberg is first-class**: Full support for time travel, schema evolution, partition evolution
- **Data products enable mesh**: Starburst data products are the interface for domain ownership
- **Caching bridges performance gaps**: Warp Speed cache compensates when source is slow
- **Lineage flows through queries**: Collibra integration tracks data lineage through Starburst

---

## Quick Reference

| Topic | Core Concept | Details |
|-------|--------------|---------|
| **Architecture** | Coordinator plans, workers execute | 6-phase execution: Parse → Plan → Schedule → Execute → Aggregate → Return |
| **Connectors** | Each has different pushdown capabilities | references/connectors.md |
| **Query Optimization** | EXPLAIN verifies pushdown | references/queries.md |
| **Security** | Authentication + Authorization + Masking | references/security.md |
| **Data Products** | Interface for data mesh domains | references/data-products.md |
| **Deployment** | Galaxy (SaaS) vs Enterprise (K8s) | references/deployment.md |
| **Monitoring** | Track P95 latency, queue time, cache hit rate | references/monitoring.md |

---

## Architecture Overview

### Coordinator-Worker Model

```
┌─────────────────────────────────────────────────────────────┐
│                      Coordinator Node                         │
│  - Query parsing, planning, optimization                      │
│  - Metadata management, catalog operations                    │
│  - Task scheduling, result aggregation                        │
│  - Client connection handling                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ Task distribution
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                      Worker Nodes                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Worker 1 │  │ Worker 2 │  │ Worker 3 │  │ Worker N │    │
│  │  - Read  │  │  - Read  │  │  - Read  │  │  - Read  │    │
│  │  - Join  │  │  - Join  │  │  - Join  │  │  - Join  │    │
│  │  - Agg   │  │  - Agg   │  │  - Agg   │  │  - Agg   │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└──────┬──────────────┬──────────────┬──────────────┬─────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Iceberg │   │   Hive   │   │PostgreSQL│   │Snowflake │
│ Connector│   │Connector │   │Connector │   │Connector │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
```

### Query Execution Phases

| Phase | Coordinator Role | Worker Role |
|-------|-----------------|-------------|
| **1. Parse** | Parse SQL, validate syntax | - |
| **2. Plan** | Generate logical plan, optimize | - |
| **3. Schedule** | Assign tasks to workers | Receive tasks |
| **4. Execute** | Monitor progress | Read data, apply filters, join, aggregate |
| **5. Aggregate** | Combine partial results | Return results to coordinator |
| **6. Return** | Return final result to client | - |

---

## Connector Ecosystem

### Connector Categories

| Category | Connectors | Use Case |
|----------|-----------|----------|
| **Data Lakes** | Iceberg, Hive, Delta Lake | Query lake tables (Bronze/Silver/Gold) |
| **RDBMS** | PostgreSQL, MySQL, SQL Server, Oracle | Join operational data with warehouse |
| **Warehouses** | Snowflake, Redshift, BigQuery | Cross-warehouse analytics |
| **Object Storage** | S3, GCS, Azure Blob | Query files directly (Parquet, ORC, JSON) |
| **Streaming** | Kafka, Kinesis | Real-time data access |
| **NoSQL** | Elasticsearch, MongoDB, Cassandra | Search and document stores |
| **SaaS** | Salesforce, Google Sheets, Airtable | Business application data |

### Iceberg Connector (First-Class)

**Capabilities:**
- Full ACID transactions (INSERT, UPDATE, DELETE, MERGE)
- Time travel and snapshot reads
- Schema evolution (add, drop, rename columns)
- Partition evolution (change partitioning without rewrite)
- Hidden partitioning (no partition filters required)
- Metadata tables for introspection
- Compaction triggers (automatic small file cleanup)

**For detailed examples and REST catalog configuration**, consult references/connectors.md covering:
- Time travel SQL examples
- Schema and partition evolution patterns
- REST catalog setup
- Metadata query patterns

### Other Connectors

**JDBC Connectors** (PostgreSQL, MySQL, SQL Server, Oracle):
- Full pushdown for filters, projections, aggregations
- Limited JOIN pushdown
- See references/connectors.md for detailed pushdown capabilities table

**Critical Pushdown Verification:**

| Operation | Push to Source? | Verify With |
|-----------|----------------|-------------|
| Filter (WHERE) | Usually yes | EXPLAIN shows filterPredicate |
| Aggregation (GROUP BY) | Usually yes | EXPLAIN shows RemoteQuery |
| JOIN | Rarely | EXPLAIN shows local join operators |

---

## Security and Access Control

### Authentication Methods

| Method | Use Case | Configuration |
|--------|----------|---------------|
| **LDAP** | Corporate directory | `authentication.type=LDAP` |
| **Active Directory** | Windows environments | `authentication.type=LDAP` (with AD config) |
| **OAuth 2.0** | Modern SSO | `http-server.authentication.type=OAUTH2` |
| **Kerberos** | Hadoop ecosystem | `http-server.authentication.type=KERBEROS` |
| **Password file** | Development only | `http-server.authentication.type=PASSWORD` |

### Authorization: Apache Ranger and OPA

**Ranger provides:**
- Database/table/column-level access control
- Row-level security (RLS) via filter functions
- Column masking based on user roles

**OPA provides:**
- Policy-as-code (Rego language)
- Fine-grained dynamic policies
- Time-based or attribute-based access

**For detailed configurations**, consult references/security.md covering:
- LDAP/OAuth/Kerberos setup
- Ranger policy examples
- Row-level security functions
- Column masking patterns
- OPA Rego policy examples

---

## Decision Guides

### Starburst vs Alternatives

| Use Case | Starburst | Snowflake | Databricks | Athena |
|----------|-----------|-----------|------------|--------|
| **Query federation** | ✅ Best | ❌ Limited | ❌ Limited | ❌ S3 only |
| **Iceberg support** | ✅ Full | ✅ Read-only | ✅ Full | ✅ Read-only |
| **Cross-warehouse** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Cost (compute)** | Pay per hour | Pay per second | Pay per DBU | Pay per query |
| **Governance** | Ranger/OPA/Collibra | Native | Unity Catalog | Lake Formation |
| **Data mesh** | ✅ Data products | ❌ No | ⚠️ Unity Catalog | ❌ No |

**When to choose Starburst:**
- Need to query across multiple systems (Iceberg + PostgreSQL + Snowflake)
- Data mesh architecture with domain ownership
- Existing Ranger/OPA governance investment
- Collibra lineage integration required

**When to choose Snowflake:**
- All data already in Snowflake
- Need stored procedures and complex SQL
- Simple governance (Snowflake RBAC sufficient)

**When to choose Databricks:**
- Heavy Spark workloads (ETL pipelines)
- ML/AI workflows (Databricks notebooks)
- Delta Lake preferred over Iceberg

**When to choose Athena:**
- Serverless, pay-per-query model
- S3-only data lake
- Infrequent queries, cost-sensitive

### Federation vs ETL

| Approach | When to Use |
|----------|-------------|
| **Federation (Starburst)** | - Ad-hoc queries across sources<br>- Source data changes frequently<br>- Don't want to duplicate data<br>- Need real-time data |
| **ETL (Airflow + Iceberg)** | - Repeated queries on same data<br>- Source data slow (latency/throughput)<br>- Complex transformations needed<br>- Guaranteed SLAs required |

**Hybrid Approach (Recommended):**
- ETL for core data products (Bronze → Silver → Gold)
- Federation for joining core with operational data
- Federation for exploratory analytics

**Example:**
```sql
-- Core data in Iceberg (ETL pipeline)
-- Operational data in PostgreSQL (live)
-- Join via federation
SELECT
    s.product_id,
    s.total_sales,
    p.current_price,
    p.inventory_level
FROM iceberg.gold.sales_summary s  -- ETL'd daily
JOIN postgresql.public.products p ON s.product_id = p.id  -- Live data
WHERE s.sale_date = CURRENT_DATE;
```

---

## Crown Equipment Context

### Starburst Role in Crown Architecture

**Position in Stack:**
- **Below**: BI tools (Tableau, Power BI), data science notebooks
- **Above**: Iceberg Bronze/Silver/Gold layers, operational databases
- **Purpose**: Query federation and access control layer

### Integration Points

| Integration | Purpose |
|-------------|---------|
| **Iceberg Lake** | Primary data source (Bronze/Silver/Gold) |
| **Collibra** | PII governance, lineage tracking |
| **Operational DBs** | Join lake with live transactional data |
| **BI Tools** | Tableau/Power BI connect to Starburst |

### Access Control Requirements

From Crown data architect review (Integration Point #8):

**Requirement:**
> Starburst access control must integrate with Collibra PII policies. When Collibra tags a column as PII, Starburst must automatically apply masking for non-privileged users.

**Implementation:**
1. Collibra tags columns as PII (e.g., customers.ssn, orders.credit_card)
2. Starburst queries Collibra API for PII tags
3. Starburst applies column masking policy based on user role
4. Admins see full values, analysts see masked values

**Configuration:**
```properties
# access-control.properties
access-control.name=collibra
collibra.api.url=https://collibra.crown.com/api
collibra.api.token=<token>
collibra.pii.tag-name=PII
collibra.masking.default-policy=PARTIAL_MASK
```

### Lineage Requirements

**Requirement:**
> All Starburst queries must send lineage events to Collibra to track data flow from Bronze → Silver → Gold → BI reports.

**Implementation:**
```properties
# catalog/iceberg.properties
iceberg.lineage.enabled=true
iceberg.lineage.endpoint=https://collibra.crown.com/api/lineage
iceberg.lineage.include-columns=true
iceberg.lineage.include-queries=true
```

**Lineage Events Captured:**
- Query text
- Source tables/columns
- Target tables/columns (for CREATE TABLE AS SELECT)
- User and timestamp
- Query execution metadata

**For detailed Collibra integration patterns**, consult references/data-products.md covering:
- Data products concept and YAML structure
- Collibra lineage architecture
- PII enforcement workflows
- Data product discovery UI

---

## Common Pitfalls

| Pitfall | Why It's Bad | Fix |
|---------|--------------|-----|
| **No pushdown verification** | Queries pull full tables | Use EXPLAIN to verify filters pushed |
| **Large JDBC table joins** | Pull full tables to Trino | Pre-aggregate in source, or ETL to Iceberg |
| **Function on partition column** | Disables partition pruning | Rewrite filter to use column directly |
| **Coordinator does data processing** | Coordinator OOM | Set `node-scheduler.include-coordinator=false` |
| **No caching for slow sources** | Every query hits slow source | Enable Warp Speed caching |
| **Undersized workers** | Out of memory on large joins | Increase worker memory or reduce data |
| **No auto-scaling** | Over-provisioned or under-provisioned | Enable HPA with CPU/memory triggers |
| **Skip Collibra integration** | Manual PII governance | Automate via Collibra policy enforcement |
| **No query monitoring** | Slow queries go unnoticed | Monitor P95 latency, set up alerts |
| **Missing connector limits** | Overwhelm source systems | Set `jdbc.max-connections`, rate limits |

---

## Detailed Reference

For in-depth guidance on specific topics, consult:

### references/connectors.md
**Covers:**
- Iceberg connector SQL examples (time travel, schema evolution)
- Iceberg REST catalog configuration
- Hive connector limitations
- JDBC connector pushdown capabilities (detailed table)
- Snowflake connector use cases
- Warp Speed caching architecture and configuration

### references/queries.md
**Covers:**
- Cross-catalog query patterns (federation examples)
- Multi-warehouse aggregation patterns
- Query optimization (EXPLAIN, CBO, partition pruning)
- Bucketing for joins
- Performance patterns (broadcast join, shuffle join, predicate/projection pushdown)
- EXPLAIN ANALYZE interpretation

### references/security.md
**Covers:**
- Authentication configuration (LDAP, AD, OAuth, Kerberos)
- Apache Ranger policy examples
- Row-level security (RLS) function patterns
- Column masking examples
- Open Policy Agent (OPA) Rego policies

### references/data-products.md
**Covers:**
- Data products concept and YAML structure
- Creating data products in Starburst
- Data product discovery UI features
- Collibra integration architecture
- PII enforcement workflows
- Lineage event capture

### references/deployment.md
**Covers:**
- Starburst Galaxy (SaaS) vs Enterprise (K8s)
- Open-source Trino comparison
- Coordinator and worker sizing tables
- Auto-scaling strategies (HPA configuration)
- Network and VPN configuration

### references/monitoring.md
**Covers:**
- Key metrics and targets
- Query history analysis SQL
- Slow query detection
- Connector performance queries
- EXPLAIN ANALYZE detailed interpretation
- Alerting queries (queue time, failures, memory pressure)
- Cache performance metrics

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Crown architecture review | ~/OneDrive/Active Centric Work/CED - Crown Equip - DA RFP - local/CED-RFP 26/governance/data-architect-review.md |
| Starburst deployment | (TBD - Kubernetes cluster) |
| Catalog configurations | (TBD - catalog/*.properties) |
| Access control policies | (TBD - access-control/*.properties) |

---

## References

- [Starburst Documentation](https://docs.starburst.io/)
- [Trino Documentation](https://trino.io/docs/current/)
- [Iceberg Connector](https://docs.starburst.io/latest/connector/iceberg.html)
- [Starburst Data Products](https://docs.starburst.io/latest/data-products/)
- [Warp Speed Caching](https://docs.starburst.io/latest/optimizer/cache.html)
- [Collibra Integration](https://docs.starburst.io/latest/security/collibra.html)
- [Query Optimization](https://trino.io/docs/current/optimizer/cost-based-optimizations.html)

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
