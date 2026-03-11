---
name: ali-collibra
description: |
  Collibra enterprise data governance platform. Use when:

  PLANNING: Designing data governance frameworks, planning Collibra Operating Model
  (Communities/Domains/Assets), architecting metadata integration patterns, evaluating
  governance workflows, considering Edge deployment for on-prem sources

  IMPLEMENTATION: Configuring Collibra Edge connectors, implementing custom asset types,
  writing REST API integrations, building lineage scanners, setting up governance workflows,
  configuring SSO/SAML authentication

  GUIDANCE: Asking about Collibra best practices, Operating Model design, Edge vs cloud
  scanning, governance workflow patterns, licensing models, SAP MDG integration

  REVIEW: Reviewing Collibra configurations, validating governance workflows, checking
  Edge connector security, auditing API integrations, verifying data quality rules

  DO NOT use for:
  - General data modeling or warehouse design (use ali-data-architecture)
  - SAP MDG-specific implementation (use ali-sap-mdg)
  - Data pipeline orchestration (use ali-apache-airflow)
  - Generic metadata management concepts (use ali-data-architecture)
---

# Collibra Enterprise Data Governance

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Collibra Operating Model (Communities, Domains, Assets, Relations)
- Planning metadata integration patterns (Edge, API, Connect)
- Architecting lineage capture strategies
- Evaluating governance workflow requirements
- Considering data quality and observability frameworks
- Planning Collibra-to-SAP MDG policy sync

**Implementation:**
- Configuring Collibra Edge for on-prem metadata scanning
- Building custom asset types and attributes
- Implementing REST API integrations
- Setting up automated lineage scanners
- Creating governance approval workflows
- Configuring Azure AD / SAML SSO
- Implementing data quality rules and scorecards

**Guidance/Best Practices:**
- Asking about Collibra Operating Model design patterns
- Asking when to use Edge vs cloud connectors
- Asking about governance workflow best practices
- Asking about licensing and sizing models
- Asking about ETL tool lineage capture
- Asking about Dev-to-Prod promotion strategies

**Review/Validation:**
- Reviewing Collibra configurations for completeness
- Validating governance workflows
- Checking Edge connector security and scheduling
- Auditing REST API integration patterns
- Verifying data quality rule effectiveness

---

## Key Principles

- **Operating Model is foundational**: Communities > Domains > Assets > Attributes > Relations hierarchy drives all governance
- **Edge for on-prem, Cloud for SaaS**: Use Collibra Edge for firewall-protected sources, cloud connectors for SaaS platforms
- **Automated lineage preferred**: Use out-of-the-box scanners (30+ supported) before custom lineage
- **Schema drift detection**: Edge scheduled scans detect and alert on metadata changes
- **Workflow configurability**: Groovy scripts enable custom approval logic and validation steps
- **Role-based licensing**: Admin, Steward/Editor, Consumer/Viewer tiers with different capabilities
- **REST API is comprehensive**: All CRUD operations on assets, attributes, relations via API v2
- **Dev-to-Prod promotion**: Workflow configurations promoted through environments
- **Quality scorecards**: Rule-based quality checks with threshold alerting and dashboards

---

## Operating Model Hierarchy

```
Organization
  └── Communities (business areas)
        └── Domains (subject areas within community)
              └── Assets (data objects: tables, columns, terms)
                    ├── Attributes (properties: description, owner, status)
                    └── Relations (links: table → schema, column → term)
```

**Example:**
```
Aliunde LLC
  └── Finance Community
        ├── Tax Practice Domain
        │     ├── Client_Master (table asset)
        │     │     ├── client_id (column asset)
        │     │     ├── tax_year (column asset)
        │     │     └── Relations:
        │     │           - maps to → "Tax Year" (business term)
        │     │           - owned by → "Tax Practice Team"
        │     └── Tax Year (business glossary term)
        │
        └── Medical EDI Domain
              ├── FACT_CLAIM (table asset)
              └── CLAIM_ID (column asset)
```

---

## Decision Guides

### When to Use Collibra vs Other Tools

| Tool Category | When to Use Collibra | When to Use Alternative |
|---------------|---------------------|------------------------|
| **vs Alation** | Enterprise governance focus, workflow heavy | Data discovery focus, lighter governance |
| **vs Informatica EDC** | Standalone governance, not tied to Informatica ETL | Already using Informatica ecosystem |
| **vs Azure Purview** | Multi-cloud, non-Azure centric | Azure-native stack only |
| **vs Apache Atlas** | Enterprise support, SaaS option | Open-source, Hadoop-centric |

### Edge vs Cloud Connectors

| Scenario | Use Edge | Use Cloud Connector |
|----------|----------|---------------------|
| Database behind firewall | ✓ | |
| Snowflake in cloud | | ✓ |
| On-prem SAP HANA | ✓ | |
| Azure SQL in cloud | | ✓ |
| Oracle on-prem | ✓ | |

---

## Common Pitfalls

| Pitfall | Why It's Bad | Do This Instead |
|---------|--------------|-----------------|
| Flat Operating Model | No logical grouping, hard to navigate | Use Communities > Domains hierarchy |
| Manual lineage only | Labor-intensive, stale quickly | Use automated scanners where possible |
| No schema drift alerts | Silent metadata changes break trust | Enable Edge drift detection |
| Overly complex workflows | Slows adoption, frustrates users | Start simple, add complexity as needed |
| No Dev environment | Test directly in Prod, risk breaking live system | Always have Dev/Test/Prod |
| Ignoring license tiers | Pay for Admin licenses when Viewer suffices | Assign roles based on actual need |
| Hardcoded domain IDs | Config breaks when promoted to Prod | Use domain names, transform on promotion |
| No quality thresholds | Rules run but no action taken | Set thresholds and alerting |
| Skipping SSO | Username/password sprawl | Integrate Azure AD / SAML early |
| No governance roles | Everyone is admin, no accountability | Define Steward, Owner, Consumer roles clearly |

---

## Detailed Reference

For implementation-level details, consult the references/ directory:

### Edge Configuration
For detailed Edge connector configuration, supported data sources, schema drift alerts, and configuration file structure, consult **references/edge-connectors.md** covering:
- Supported JDBC data sources (MySQL, Oracle, PostgreSQL, SAP HANA, Snowflake, SQL Server, etc.)
- Edge capabilities (catalog connectors, lineage connectors, profiling, scheduled scans)
- Complete edge-config.yaml pattern with connection and scanner definitions
- Schema drift detection configuration
- Deployment models (Cloud SaaS, On-Premises, Hybrid)

### Lineage Implementation
For automated lineage scanner details and ETL tool integration, consult **references/lineage-scanners.md** covering:
- Out-of-the-box scanners for 30+ platforms (SQL dialects, ETL tools, BI tools)
- Lineage capture pattern and SQL AST parsing
- Complete SQL lineage example (Snowflake INSERT...SELECT)
- Decision guide for automated vs manual lineage

### REST API Development
For REST API integration code and endpoint reference, consult **references/rest-api-integration.md** covering:
- Complete CollibraClient Python class with CRUD methods
- Asset creation, attribute management, relation creation
- Search functionality
- Full REST API endpoint quick reference (/assets, /attributes, /relations, /domains, etc.)

### Custom Asset Types
For creating custom asset types and attributes, consult **references/custom-asset-types.md** covering:
- Asset type definition YAML pattern (example: API Endpoint)
- Python code for creating asset types via REST API
- Attribute type configuration (picklists, text, numeric)
- Core modules overview (Data Catalog, Business Glossary, Data Lineage, etc.)

### Workflow Configuration
For governance workflow implementation, consult **references/workflow-implementation.md** covering:
- Complete workflow structure YAML (Data Asset Approval example)
- Dev-to-Prod promotion pattern and environment strategy
- Custom Groovy script examples for validation logic
- Step definitions (auto-assign, human approval, status updates, notifications)

### Data Quality Rules
For data quality and scorecard implementation, consult **references/data-quality-rules.md** covering:
- Quality rule definition YAML (format validation, threshold alerting)
- Quality scorecard configuration (weighted dimensions, rule assignments)
- SQL-based validation logic examples
- Schedule and alert configuration

### Integration Examples
For specific platform integrations, consult **references/integration-examples.md** covering:
- Collibra-to-SAP MDG policy sync implementation
- Azure AD / SAML SSO configuration YAML
- Informatica PowerCenter lineage extraction
- Azure Data Factory lineage extraction
- Complete integration code examples

### Implementation Strategies
For project rollout patterns, consult **references/implementation-patterns.md** covering:
- Greenfield implementation timeline (4-phase, 16-week plan)
- Brownfield migration pattern (5-step process)
- Phased rollout by domain strategy
- Success criteria and timeline guidance

### Environment Management
For DevOps patterns and configuration promotion, consult **references/environment-management.md** covering:
- Environment strategy (Dev, Test, Prod refresh frequencies)
- Configuration promotion Python code
- ID transformation functions for cross-environment promotion

### Licensing and Sizing
For procurement and capacity planning, consult **references/licensing-and-sizing.md** covering:
- License tier capabilities (Admin, Steward/Editor, Consumer/Viewer)
- Sizing guidance table (small/medium/large deployments)
- User count, Edge agent, and database recommendations

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Core skill content | skills/ali-collibra/SKILL.md |
| Edge connector details | skills/ali-collibra/references/edge-connectors.md |
| Lineage scanner details | skills/ali-collibra/references/lineage-scanners.md |
| REST API integration | skills/ali-collibra/references/rest-api-integration.md |
| Custom asset types | skills/ali-collibra/references/custom-asset-types.md |
| Workflow implementation | skills/ali-collibra/references/workflow-implementation.md |
| Data quality rules | skills/ali-collibra/references/data-quality-rules.md |
| Integration examples | skills/ali-collibra/references/integration-examples.md |
| Implementation patterns | skills/ali-collibra/references/implementation-patterns.md |
| Environment management | skills/ali-collibra/references/environment-management.md |
| Licensing and sizing | skills/ali-collibra/references/licensing-and-sizing.md |

---

## References

- [Collibra Documentation](https://productresources.collibra.com/docs/)
- [Collibra REST API v2](https://developer.collibra.com/rest/api/)
- [Collibra Edge Documentation](https://productresources.collibra.com/docs/edge/)
- [Collibra Marketplace](https://marketplace.collibra.com/)
- [Azure AD SAML Integration](https://productresources.collibra.com/docs/saml-sso/)

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
