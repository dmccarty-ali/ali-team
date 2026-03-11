# ali-starburst - Data Products and Governance

Starburst Data Products concept and Collibra integration details.

---

## Data Products Concept

Data products are the **interface** for domain ownership in data mesh architecture.

### YAML Structure

```yaml
# data_product.yaml
name: customer_360
owner: customer_analytics_team
description: Unified customer view with demographics and behavior
domain: customer

tables:
  - iceberg.gold.dim_customer
  - iceberg.gold.fact_customer_interactions
  - iceberg.gold.customer_lifetime_value

sla:
  freshness: 1 hour
  uptime: 99.9%
  query_latency_p95: 5s

access:
  public_read: true
  pii_governance: collibra_policy_id_42

lineage:
  upstream:
    - iceberg.silver.crm_contacts
    - iceberg.silver.web_events
    - postgresql.public.orders
```

---

## Creating Data Products

### SQL-Based Configuration

```sql
-- Tag tables as part of data product
ALTER TABLE iceberg.gold.dim_customer
SET PROPERTIES domain = 'customer_analytics';

-- Define SLAs and ownership
ALTER TABLE iceberg.gold.dim_customer
SET PROPERTIES
    owner = 'customer_team@example.com',
    sla_freshness = '1 hour',
    sla_uptime = '99.9%';
```

### Multi-Table Data Products

```sql
-- Tag multiple tables as belonging to same data product
ALTER TABLE iceberg.gold.dim_customer
SET PROPERTIES data_product = 'customer_360';

ALTER TABLE iceberg.gold.fact_customer_interactions
SET PROPERTIES data_product = 'customer_360';

ALTER TABLE iceberg.gold.customer_lifetime_value
SET PROPERTIES data_product = 'customer_360';
```

---

## Data Product Discovery

Starburst Enterprise includes a data product catalog UI where:

### Discovery Features

- **Browse by domain**: Users explore data products organized by business domain
- **Published documentation**: Owners publish schemas, usage examples, SLA commitments
- **SLA monitoring**: Freshness, uptime, and performance metrics visualized in real-time
- **Access requests**: Self-service request workflow with owner approval
- **Lineage visualization**: Interactive graph showing upstream dependencies
- **Quality metrics**: Data quality scores and validation results

### Search Capabilities

- Full-text search across product names, descriptions, column names
- Filter by domain, owner, SLA tier, access level
- Tag-based discovery (PII, critical, experimental)

---

## Collibra Integration

### Integration Points

| Integration | Purpose |
|-------------|---------|
| **Lineage** | Track data flow through Starburst queries |
| **Governance** | Enforce Collibra policies in Starburst |
| **PII Protection** | Mask PII columns based on Collibra tags |
| **Data Quality** | Validate data against Collibra quality rules |

### Lineage Architecture

```
Starburst Query
     │
     ▼
Query Execution
     │
     ▼
Lineage Events (catalog, schema, table, column)
     │
     ▼
Starburst Lineage Collector
     │
     ▼
Collibra API
     │
     ▼
Collibra Data Lineage Graph
```

### Configuration

```properties
# catalog/iceberg.properties
# Enable lineage tracking
iceberg.lineage.enabled=true
iceberg.lineage.endpoint=https://collibra.example.com/api
iceberg.lineage.api-key=<key>
iceberg.lineage.include-columns=true
iceberg.lineage.include-queries=true
```

### Lineage Events Captured

- Query text
- Source tables/columns
- Target tables/columns (for CREATE TABLE AS SELECT)
- User and timestamp
- Query execution metadata

### PII Enforcement via Collibra

**Workflow:**
1. Data steward tags column as PII in Collibra (e.g., customers.ssn)
2. Collibra policy: "PII columns require masking for non-admin users"
3. Starburst queries Collibra API for PII tags
4. Starburst applies masking policy automatically

**Example Query:**
```sql
-- User queries table with PII column
SELECT customer_id, ssn, email FROM iceberg.gold.customers;

-- Starburst checks Collibra:
--   - ssn tagged as PII? Yes
--   - User role? analyst (not admin)
--   - Policy: mask PII for analysts

-- Result set masks SSN:
-- customer_id | ssn          | email
-- ------------|--------------|------------------
-- C001        | XXX-XX-1234  | customer@example.com
```

### Collibra Configuration

```properties
# access-control.properties
access-control.name=collibra
collibra.api.url=https://collibra.example.com/api
collibra.api.token=<token>
collibra.pii.tag-name=PII
collibra.masking.default-policy=PARTIAL_MASK
```

---

**Last Updated:** 2026-02-16
