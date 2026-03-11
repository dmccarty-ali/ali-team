# ali-data-architecture - Data Mesh Concepts Reference

Federated data ownership with domain data products.

---

## Core Concepts

| Concept | Description | Example |
|---------|-------------|---------|
| **Domain Ownership** | Teams own their data | Claims team owns claims data product |
| **Data as Product** | Data treated as product with SLAs | Claims API with uptime, latency guarantees |
| **Self-serve Platform** | Infrastructure enables teams | Shared Snowflake, Airflow, catalog |
| **Federated Governance** | Global policies, local implementation | PII standards enforced per domain |

---

## Data Product Definition

```yaml
# data_product.yaml
name: claims_data_product
domain: claims_processing
owner: claims-team@aliunde.com
description: Processed healthcare claims with payment status

schema:
  database: EDW_MEDEDI
  tables:
    - FACT_CLAIM
    - DIM_PROVIDER
    - DIM_PAYER

sla:
  freshness: 4 hours  # Data updates within 4 hours
  uptime: 99.5%       # 99.5% availability
  support_tier: P2    # Support response time

access:
  public: true
  pii_fields:
    - FACT_CLAIM.patient_id
    - DIM_PROVIDER.provider_npi

lineage:
  sources:
    - LAKE_MEDEDI.EDI_835_RAW
    - LAKE_MEDEDI.EDI_837I_RAW
```

---

## Principles

1. **Domain teams own end-to-end**: Ingestion, transformation, publishing
2. **Data products have APIs**: SQL views, REST endpoints, event streams
3. **Platform provides infrastructure**: Compute, storage, orchestration, catalog
4. **Global governance policies**: Security, privacy, quality standards
5. **Federated execution**: Each domain implements within standards

---

## Benefits

- Scalability: Decentralized ownership, no bottleneck
- Quality: Domain experts closest to data
- Agility: Teams ship independently
- Discovery: Catalog of data products

---

## Challenges

- Consistency across domains
- Duplicate infrastructure
- Cross-domain coordination
- Governance enforcement
