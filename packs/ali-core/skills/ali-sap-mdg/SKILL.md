---
name: ali-sap-mdg
description: |
  SAP Master Data Governance (MDG) patterns for enterprise master data management. Use when:

  PLANNING: Designing MDG hub vs co-deployment architecture, planning domain rollout strategy,
  architecting matching and consolidation rules, considering integration patterns for multi-system
  landscapes, evaluating BRFplus workflow design

  IMPLEMENTATION: Building BRFplus decision tables, implementing matching rules and survivorship logic,
  configuring change request workflows, setting up MDI/IDoc integration, extending data models for
  custom domains, implementing DQM address validation

  GUIDANCE: Asking about hub vs co-deployment trade-offs, BRFplus workflow configuration best practices,
  matching engine tuning, MDI vs ALE/IDoc integration choices, transport management strategy, domain
  implementation timelines

  REVIEW: Reviewing workflow decision tables for completeness, validating matching rule accuracy,
  checking survivorship logic, auditing integration configurations, verifying transport setup

  Do NOT use for:
  - Generic data architecture without SAP MDG context (use ali-data-architecture)
  - General ERP implementation without master data governance focus
  - Data governance strategy without SAP-specific implementation (use ali-data-architecture)
  - Non-SAP MDM products (Informatica, Talend, etc.)
  - Simple data quality without governance workflows
---

# SAP Master Data Governance (MDG)

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing SAP MDG architecture (hub vs co-deployment)
- Planning domain rollout strategy and prioritization
- Architecting matching and consolidation rules
- Designing BRFplus workflow routing
- Evaluating integration patterns for multi-system landscapes
- Planning transport management and client strategy

**Implementation:**
- Configuring BRFplus decision tables (DT_SINGLE_VAL, DT_USER_AGT_GRP, DT_CONDITIONS)
- Building matching rules and survivorship logic
- Implementing change request workflows via USMD_SSW_RULE
- Setting up SAP Master Data Integration (MDI) or ALE/IDoc distribution
- Extending data models for custom domains
- Implementing Data Quality Management (DQM) validation

**Guidance/Best Practices:**
- Asking about hub vs co-deployment trade-offs
- Asking how to configure BRFplus workflows
- Asking about matching engine tuning (deterministic vs probabilistic)
- Asking which integration pattern to use (MDI, ALE/IDoc, REST/OData)
- Asking about typical implementation timelines
- Asking about transport management best practices

**Review/Validation:**
- Reviewing workflow decision tables for completeness
- Validating matching rule accuracy and performance
- Checking survivorship logic for business correctness
- Auditing integration configurations for security and reliability
- Verifying transport setup across DEV/QA/PROD landscape

---

## Key Principles

- **Hub is for complexity**: Co-deployment works for simple single-domain scenarios; hub deployment scales for multi-domain, multi-system landscapes
- **Business Partner is the future**: S/4HANA uses Business Partner (BP) for customer/vendor master, not legacy KNA1/KNB1
- **BRFplus controls workflows**: MDG workflows use BRFplus decision tables (NOT conventional SAP Workflow), configured via USMD_SSW_RULE
- **Matching requires tuning**: Out-of-box matching is a starting point; production requires tuning thresholds and steward training
- **Survivorship rules are business decisions**: Which source wins per attribute is a business rule, not a technical decision
- **MDI is cloud-native**: SAP Master Data Integration (MDI) is the modern integration layer; ALE/IDoc is legacy but still common
- **Golden record is computed**: Best record is calculated from survivorship rules, not manually curated
- **Transport via TMS**: Standard SAP Transport Management System (STMS); MDG customizing follows standard SAP transport patterns

---

## Platform Architecture

### Deployment Models

| Model | Architecture | Use Case | Complexity |
|-------|--------------|----------|------------|
| **Co-Deployment** | MDG activated in existing S/4HANA transactional system | Single domain (e.g., Customer only), simple governance | Low |
| **Hub Deployment** | Dedicated S/4HANA instance for master data governance only | Multi-domain, multi-system, complex consolidation | High |

### Hub vs Co-Deployment Trade-offs

**Co-Deployment:**
- ✅ Lower infrastructure cost (one S/4HANA instance)
- ✅ Simpler architecture (no synchronization needed)
- ✅ Faster implementation for single domain
- ❌ Transactional load competes with governance processes
- ❌ Limited scalability for multiple domains
- ❌ Harder to govern multiple source systems

**Hub Deployment:**
- ✅ Dedicated resources for governance workloads
- ✅ Scales to multiple domains and source systems
- ✅ Central golden record repository
- ✅ Isolates governance from transactional systems
- ❌ Higher infrastructure cost (dedicated S/4HANA)
- ❌ Requires integration to distribute golden records
- ❌ More complex to implement and maintain

**Decision Guide:**
- **Choose Co-Deployment** if: Single domain (Customer OR Material), single S/4HANA source, <10K records, simple workflows
- **Choose Hub** if: Multiple domains (Customer AND Material AND Supplier), multiple source systems, >50K records, complex matching/consolidation

### Available Platforms

| Platform | Deployment | Availability |
|----------|------------|--------------|
| **S/4HANA On-Prem** | Hub or co-deployment | Version 2021 or later |
| **S/4HANA Cloud Private** | Hub or co-deployment | RISE with SAP, Private Cloud Edition |
| **SAP BTP** | Cloud-native MDG apps | Limited (not full MDG yet) |

**Note:** MDG is NOT available in S/4HANA Cloud Public Edition (multi-tenant SaaS). Public cloud customers use SAP Master Data Governance on SAP BTP (limited functionality) or RISE Private Cloud.

---

## Data Domains

### Core Domains

| Domain | S/4HANA Object | Tables | Use Case |
|--------|----------------|--------|----------|
| **Customer Master** | Business Partner (BP) | BUT000, BUT020, BUT050 | Customer consolidation, hierarchy management |
| **Material Master** | Material | MARA, MARC, MARD | Product master, BOM management, classification |
| **Supplier/Vendor** | Business Partner (BP) | BUT000, LFA1 (legacy) | Supplier onboarding, risk management |
| **Finance Master** | G/L Account, Cost Center, Profit Center | SKA1, CSKS, CEPC | Chart of accounts, cost accounting |
| **Custom Objects** | Extensible data model | Z-tables, custom entities | Non-standard domains (contracts, assets) |

**For detailed domain complexity timelines and Business Partner migration patterns, consult references/implementation-details.md covering:**
- Business Partner vs legacy table migration
- Domain complexity estimates and typical timelines
- Greenfield vs brownfield implementation patterns
- Phased domain rollout recommendations

---

## Matching and Consolidation

### Matching Engine

**Two matching approaches:**

| Approach | How It Works | Use Case |
|----------|--------------|----------|
| **Deterministic** | Exact match on business keys (e.g., Tax ID, Email) | High-confidence matching, structured data |
| **Probabilistic** | Fuzzy matching with scoring (name similarity, address) | Low-quality data, missing keys, manual review |

### Matching Thresholds

| Threshold | Action | Typical Range |
|-----------|--------|---------------|
| **Auto-Match** | Automatic merge without review | 95-100% |
| **Manual Review** | Send to steward for adjudication | 70-94% |
| **No Match** | Treat as unique record | 0-69% |

**Best Practice:** Start conservative (higher thresholds), tune based on false positive/negative analysis.

### Survivorship Rules Overview

**Which source wins per attribute when consolidating duplicates:**

- Priority-based (e.g., ERP_GOLD > CRM > Salesforce)
- Attribute-specific authority (e.g., only GOVERNMENT_REGISTRY updates TAX_ID)
- Data quality-based (prefer DQM-validated addresses)
- Time-based (most recent update wins)
- Business rule-based (complex conditional logic)

**For detailed match configuration code examples, survivorship rule patterns, and DQM integration specifics, consult references/matching-configuration.md covering:**
- Deterministic and probabilistic match configuration code
- Survivorship rule examples (priority, authority, quality-based)
- DQM address validation configuration
- Manual review queue process
- Matching performance tuning and metrics

---

## Governance Workflows (BRFplus-Based)

### Workflow Architecture

**MDG uses BRFplus (Business Rule Framework plus), NOT conventional SAP Workflow.**

**Key Transaction:** USMD_SSW_RULE - Configure workflow decision tables

**Change Request Workflow:**
```
1. User submits change request
   ↓
2. BRFplus evaluates DT_SINGLE_VAL_<type> (process flow)
   ↓
3. BRFplus evaluates DT_USER_AGT_GRP_<type> (who approves)
   ↓
4. Approval step(s)
   ↓
5. BRFplus evaluates DT_CONDITIONS_<type> (conditional routing)
   ↓
6. Final activation (write to tables)
```

### Three Decision Tables per Change Request Type

| Decision Table | Purpose | Configuration |
|----------------|---------|---------------|
| **DT_SINGLE_VAL_\<type\>** | Process flow between steps | Define step sequence (Create → Validate → Approve → Activate) |
| **DT_USER_AGT_GRP_\<type\>** | Processor/agent determination | Who receives work items at each step (by role, attribute value) |
| **DT_CONDITIONS_\<type\>** | Conditional routing | IF/THEN rules (if amount > $10K, require VP approval) |

**For detailed BRFplus decision table structures, change request types, and code examples, consult references/workflow-configuration.md covering:**
- Three decision tables detail (structure, configuration)
- Change request types (Create, Change, Block, Delete, Mass)
- Example BRFplus configurations (credit limit approval, agent determination by geography)
- Workflow testing checklist
- Common configuration mistakes

---

## Integration Patterns

### SAP Master Data Integration (MDI)

**Cloud-native integration service (recommended for new implementations):**
- Real-time or near-real-time replication
- REST/OData APIs for non-SAP systems
- Pre-built connectors for SAP systems (S/4HANA, ECC, CRM)
- Handles transformation and mapping
- Part of SAP Integration Suite

**Use When:**
- Integrating with cloud systems
- Need real-time data distribution
- Modernizing from ALE/IDoc
- Multi-cloud architecture

### ALE/IDoc (Legacy but Common)

**Traditional SAP-to-SAP distribution:**
- IDoc messages for master data distribution
- ALE (Application Link Enabling) for routing
- Near-real-time or batch processing
- Well-established, stable, but aging

**Use When:**
- Existing ALE/IDoc landscape
- SAP-to-SAP only (no cloud systems)
- Batch distribution acceptable
- Low budget for modernization

**For detailed integration configuration examples, OData API usage, and Kafka event-driven patterns, consult references/integration-patterns.md covering:**
- MDI capabilities and flow examples
- ALE/IDoc common types and processing flow
- REST/OData API examples (query, create, update)
- Kafka integration for event-driven architectures
- Integration pattern comparison and monitoring

---

## Common Pitfalls

| Pitfall | Why It's Bad | Prevention |
|---------|--------------|------------|
| **Underestimating matching tuning** | Out-of-box matching produces 40%+ false positives | Budget 3-4 weeks for match rule tuning with real data |
| **Ignoring survivorship rules** | Produces incorrect golden records | Define survivorship rules BEFORE implementation starts |
| **Skipping DQM integration** | Poor data quality propagates downstream | Include DQM in scope from Day 1 |
| **Not training stewards** | Manual review queue backs up, poor decisions | Train stewards on matching concepts and UI before go-live |
| **Over-complex workflows** | Approval delays, user frustration | Start simple (2-step approval), add complexity later |
| **Neglecting performance testing** | Slow matching/consolidation on large datasets | Test with production-scale data in QA (>100K records) |
| **Assuming Business Partner = Customer** | BP is also vendor, prospect, partner (multi-role) | Design for multi-role from start |
| **Ignoring transport dependencies** | Failed transports in QA/PROD | Document transport sequence, test in sandbox first |
| **No rollback plan** | Stuck if go-live fails | Always have rollback plan (revert transports, restore data) |

---

## Decision Guides

### When to Implement MDG

**Good candidates:**
- Multiple source systems with duplicate/conflicting master data
- M&A scenarios (consolidating acquired companies)
- Data quality issues causing operational problems
- Regulatory compliance requiring golden record audit trail
- Customer 360 or supplier risk management initiatives

**Not good candidates:**
- Single ERP system with clean data
- Small companies (<$100M revenue, <50K records)
- No dedicated data steward resources
- Budget <$500K (too complex for low budget)

### Hub vs Co-Deployment Decision Tree

```
Start: Do you have multiple domains (Customer AND Material AND Supplier)?
├─ YES → Do you have multiple source systems?
│   ├─ YES → Hub Deployment
│   └─ NO → Do you have >50K records per domain?
│       ├─ YES → Hub Deployment
│       └─ NO → Co-Deployment acceptable
└─ NO (single domain) → Do you need consolidation from multiple sources?
    ├─ YES → Hub Deployment
    └─ NO → Co-Deployment
```

### Domain Prioritization

**Prioritize by business impact and complexity:**

| Domain | Business Impact | Complexity | Priority |
|--------|----------------|------------|----------|
| Supplier | High (risk/compliance) | Low | 1st |
| Customer | High (revenue) | Medium | 2nd |
| Material | Medium (operations) | High | 3rd |
| Finance | Medium (reporting) | Medium | 4th |

**Start with highest impact, lowest complexity** (quick win).

### Matching Strategy Selection

| Data Quality | Recommended Approach |
|--------------|---------------------|
| High (clean keys) | Deterministic only (faster, cheaper) |
| Medium (some missing keys) | Deterministic + probabilistic with manual review |
| Low (poor quality) | Probabilistic + extensive manual review + DQM cleansing |

---

## Quick Reference

### Key Transactions

| Transaction | Purpose |
|-------------|---------|
| **USMD_SSW_RULE** | Configure BRFplus workflow decision tables |
| **USMD_RULE** | Configure matching and validation rules |
| **USMD_MODEL** | Data model configuration |
| **NWBC** | Launch MDG UI (Fiori Launchpad in modern systems) |
| **DQM_MAIN** | Configure Data Quality Management |
| **STMS** | Transport Management System |
| **SE09/SE10** | Transport Organizer |
| **SLG1** | Application log (troubleshooting) |

### BRFplus Decision Tables

```
For each Change Request Type:
- DT_SINGLE_VAL_<type>    Process flow (step sequence)
- DT_USER_AGT_GRP_<type>  Agent determination (who approves)
- DT_CONDITIONS_<type>    Conditional routing (if/then rules)
```

### Common Tables

| Table | Purpose |
|-------|---------|
| **USMD210** | Change request header |
| **USMD211** | Change request items |
| **BUT000** | Business Partner header |
| **MARA** | Material master (general) |
| **MARC** | Material master (plant) |

---

## Detailed References

For deep-dive technical content, consult these reference files:

### Implementation & Planning
- **references/implementation-details.md** - Greenfield/brownfield patterns, phased rollout, dual-ERP scenarios, timeline estimates, project structure templates

### Configuration & Development
- **references/workflow-configuration.md** - BRFplus decision table structures, change request types, code examples for credit limit approval and agent determination
- **references/matching-configuration.md** - Match configuration code (deterministic, probabilistic, hybrid), survivorship rule examples, DQM integration, manual review queue process, performance tuning
- **references/integration-patterns.md** - MDI capabilities and flow, ALE/IDoc configuration, REST/OData API examples, Kafka event-driven patterns, integration monitoring

### Operations & Quality
- **references/data-quality.md** - DQM integration points, validation rule code examples, address standardization scenarios, batch cleansing jobs, duplicate detection, quality metrics
- **references/environment-management.md** - Transport management details, client strategy (3-tier vs 4-tier), deployment checklist, go-live procedure, rollback plan

### Business & Sizing
- **references/licensing-sizing.md** - SAP MDG licensing details (included in S/4HANA, not ECC), platform availability, sizing considerations (CPU, RAM, storage), cost estimation, rightsizing recommendations
- **references/project-structure.md** - Recommended directory structure, configuration file organization, documentation templates, version control strategy, maintenance activities

---

## References

- [SAP MDG for S/4HANA Documentation](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/master-data-governance)
- [BRFplus Configuration Guide](https://help.sap.com/docs/SAP_NETWEAVER_750/bdd5a6a5da124c7e9c13407af00a5e8b/4e1e5cd24c9c4c55e10000000a42189b.html)
- [SAP Master Data Integration (MDI)](https://help.sap.com/docs/SAP_MASTER_DATA_INTEGRATION)
- [Business Partner Migration S/4HANA](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/business-partner)
- [SAP Data Quality Management](https://help.sap.com/docs/SAP_DATA_QUALITY_MANAGEMENT)

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
