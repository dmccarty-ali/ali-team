---
name: ali-data-governance
description: |
  Data governance program design - operating models, stewardship frameworks, data domains,
  CDE identification, maturity assessments, and DAMA-DMBOK alignment. Use when:

  PLANNING: Designing governance operating models (centralized/federated/hybrid), planning
  stewardship frameworks, identifying data domains, scoping CDE programs, architecting
  governance councils, designing data classification schemes

  IMPLEMENTATION: Defining governance RACI matrices, building CDE inventories, writing
  business term definitions, designing quality rule frameworks, creating maturity roadmaps,
  documenting naming conventions and standards

  GUIDANCE: Asking about governance operating model selection, stewardship role design,
  CDE prioritization methodology, DAMA-DMBOK knowledge areas, maturity model approaches,
  data classification best practices, governance program structure

  REVIEW: Reviewing governance program designs, auditing stewardship frameworks, validating
  RACI completeness, checking CDE identification rigor, assessing maturity assessments,
  evaluating data quality framework design

  DO NOT use for:
  - Collibra platform configuration (use ali-collibra)
  - SAP MDG tool configuration (use ali-sap-mdg)
  - Data pipeline design (use ali-apache-airflow or ali-data-architecture)
  - Data warehouse modeling (use ali-data-architecture)
---

# Data Governance Program Design

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Selecting a governance operating model for an organization
- Designing stewardship role structures and responsibility frameworks
- Identifying and bounding data domains
- Scoping critical data element programs
- Architecting governance councils and steering committees
- Designing data classification schemes

**Implementation:**
- Building governance RACI matrices for key activities
- Writing CDE definitions and inventories
- Creating business term definition standards
- Designing data quality rule frameworks (pre-tool)
- Building maturity assessment worksheets
- Documenting naming conventions and metadata standards

**Guidance/Best Practices:**
- Which governance operating model fits our organization?
- How do we define data steward roles without burning people out?
- How do we identify and prioritize critical data elements?
- What does DAMA-DMBOK say about this?
- How do we assess our current governance maturity?
- How do we handle data classification across regulated data types?

**Review/Validation:**
- Reviewing governance program designs for completeness
- Auditing stewardship frameworks for accountability gaps
- Validating RACI matrices for coverage
- Checking CDE identification rigor
- Assessing maturity assessment methodology
- Evaluating data quality framework design before tool selection

---

## Key Principles

- **Operating model before tools**: Choose governance structure before selecting Collibra, Purview, or any catalog tool
- **Business ownership is non-negotiable**: IT facilitates governance; business owns it
- **Start federated, formalize hybrid**: Most orgs start federated organically; formalize with hybrid as maturity grows
- **CDEs are the lever**: Focus quality and stewardship effort on the 5-15% of data that drives 80% of business decisions
- **RACI gaps kill programs**: Every governance activity must have exactly one Accountable owner
- **Maturity is a journey, not a destination**: Target Level 3 (Defined) before attempting Level 4/5
- **DMBOK is a framework, not a prescription**: Apply the relevant knowledge areas; not all 11 areas need equal investment
- **Quality is a business problem**: Data quality rules must be designed by business before any tool is configured
- **Governance council authority matters**: Without executive sponsorship and decision rights, governance is advisory only

---

## 1. Governance Operating Models

### Three Model Types

| Dimension | Centralized | Federated | Hybrid |
|-----------|-------------|-----------|--------|
| **Decision Authority** | Central governance team | Domain/business unit | Shared: enterprise policy + domain execution |
| **Policy Creation** | Top-down | Bottom-up | Enterprise core policies + domain extensions |
| **Steward Location** | Central team | Embedded in business | Embedded with central coordination |
| **Speed** | Slow (bottleneck risk) | Fast (inconsistency risk) | Balanced |
| **Consistency** | High | Low | Medium-High |
| **Best For** | Highly regulated industries, small orgs | Large enterprises, autonomous BUs | Most mature enterprise programs |
| **Crown Equipment Fit** | No (too large, distributed) | Partial (current state) | Yes (target state) |

### Decision Matrix: Selecting an Operating Model

| Factor | Points to Centralized | Points to Federated | Points to Hybrid |
|--------|----------------------|---------------------|------------------|
| Organization size | Small (<500 employees) | Large, multiple BUs | Mid-to-large with shared data |
| Regulatory environment | High (HIPAA, SOX, GDPR) | Low | Mixed by domain |
| Data sharing across BUs | High (shared CDEs) | Low (BU-isolated) | Medium (some cross-domain |
| Current governance maturity | Level 1-2 (needs structure) | Level 3+ (already organized) | Level 2-3 (transitioning) |
| Executive sponsorship | CDAO/CDO exists | No central data leader | CDO with domain owners |
| Culture | Command-and-control | Autonomous BUs | Collaborative |

### Organizational Structures

**Centralized Model:**
```
Chief Data Officer
└── Data Governance Office
      ├── Data Governance Manager
      ├── Data Stewards (central pool)
      └── Data Quality Analysts
```

**Federated Model:**
```
No Central Authority
├── Finance Domain Owner (VP Finance) + Finance Data Stewards
├── Supply Chain Domain Owner (VP Ops) + SC Data Stewards
└── Customer Domain Owner (VP Sales) + Customer Data Stewards
```

**Hybrid Model (recommended for most enterprises):**
```
Chief Data Officer
├── Data Governance Council (domain owners + CDO)
├── Enterprise Data Governance Office (policy, standards, tools)
│     ├── Data Governance Lead
│     ├── Metadata/Catalog Manager
│     └── Data Quality Manager
├── Finance Domain (VP Finance = Domain Owner)
│     └── Finance Data Stewards (embedded)
├── Supply Chain Domain (VP Ops = Domain Owner)
│     └── SC Data Stewards (embedded)
└── Customer Domain (VP Sales = Domain Owner)
      └── Customer Data Stewards (embedded)
```

### Pros and Cons Summary

| Model | Pros | Cons |
|-------|------|------|
| **Centralized** | Consistent standards, clear accountability, fast policy changes | Bottleneck risk, low business buy-in, doesn't scale |
| **Federated** | Business ownership, agility, domain expertise | Inconsistency, duplication, no cross-domain view |
| **Hybrid** | Balance of consistency and agility, scalable, DMBOK-aligned | More complex to operate, requires mature coordination |

---

## 2. DAMA-DMBOK Framework

### 11 Knowledge Areas

| # | Knowledge Area | Core Activities | Typical Maturity Starts |
|---|---------------|-----------------|------------------------|
| 1 | Data Governance | Policy, roles, accountability, standards | Level 1 (ad hoc) |
| 2 | Data Architecture | Models, standards, integration patterns | Level 1-2 |
| 3 | Data Modeling & Design | Logical/physical models, naming conventions | Level 2 |
| 4 | Data Storage & Operations | Database admin, backup, performance | Level 2-3 |
| 5 | Data Security | Classification, access control, masking | Level 1-2 |
| 6 | Data Integration & Interoperability | ETL/ELT, APIs, data sharing | Level 2 |
| 7 | Document & Content Management | Unstructured data, records management | Level 1 |
| 8 | Reference & Master Data | MDM, reference data management | Level 1-2 |
| 9 | Data Warehousing & BI | Warehouses, marts, reporting | Level 2-3 |
| 10 | Metadata Management | Catalog, lineage, business glossary | Level 1-2 |
| 11 | Data Quality | Profiling, rules, remediation, monitoring | Level 1-2 |

### Phased DMBOK Alignment Approach

| Phase | Timeline | Focus Areas | Deliverables |
|-------|----------|-------------|--------------|
| **Phase 1: Foundation** | Months 1-3 | Data Governance (#1), Data Security (#5) | Operating model, RACI, classification scheme |
| **Phase 2: Visibility** | Months 3-6 | Metadata Management (#10), Reference & Master Data (#8) | Business glossary, CDE inventory, MDM scope |
| **Phase 3: Quality** | Months 6-12 | Data Quality (#11), Data Integration (#6) | Quality rules, DQ scorecard, integration standards |
| **Phase 4: Maturity** | Months 12-24 | Data Architecture (#2), Data Modeling (#3), Data Warehousing (#9) | Architecture standards, model governance, BI governance |
| **Phase 5: Optimization** | Ongoing | All areas at Level 3+ | Automation, continuous improvement, ML governance |

---

## 3. Stewardship Frameworks

### Role Definitions Table

| Role | Business or IT | Accountability | Time Commitment | Decision Authority |
|------|---------------|----------------|-----------------|-------------------|
| **Data Domain Owner** | Business (VP/Director level) | Accountable for all data in domain | 5-10% of time | Approve CDEs, resolve cross-domain disputes, fund quality remediation |
| **Data Steward** | Business (SME level) | Responsible for day-to-day governance activities | 15-25% of time | Define business terms, approve quality rules, triage data issues |
| **Data Custodian** | IT (DBA/Engineer) | Responsible for technical management of data | Varies | Implement security controls, manage access, execute quality fixes |
| **Data Consumer** | Business or IT | Uses data for analytics, reporting, decisions | N/A | Request access, report quality issues |
| **Governance Council** | Business + IT (senior) | Accountable for program direction and policy | Monthly meetings | Approve enterprise policies, resolve escalations, prioritize roadmap |

### Stewardship RACI for Core Activities

| Activity | Domain Owner | Data Steward | Data Custodian | Governance Council | IT/Engineering |
|----------|-------------|-------------|---------------|-------------------|----------------|
| Define CDE | A | R | C | I | I |
| Approve business term | A | R | I | I | I |
| Create quality rule | A | R | C | I | I |
| Implement quality rule | I | C | R | I | A |
| Resolve data issue (<5 days) | I | A/R | C | I | C |
| Resolve data issue (>5 days) | A | R | C | I | R |
| Approve data access request | A | C | R | I | I |
| Escalate cross-domain conflict | C | I | I | A/R | I |
| Approve enterprise data policy | C | I | I | A/R | I |
| Update data classification | A | R | C | I | I |

R = Responsible, A = Accountable, C = Consulted, I = Informed

### Steward Selection Criteria

- Subject matter expertise in the data domain (knows the data, not just uses it)
- Credibility with peers (respected by data consumers and domain owner)
- Collaborative mindset (governance requires negotiation, not mandate)
- Available bandwidth (15-25% minimum; failure mode is stewards with 0% allocated time)
- Longevity in role (avoid frequent rotation; institutional knowledge is the asset)

### Escalation Path

```
Data Consumer reports issue
    ↓
Data Steward triages and resolves (target: 5 business days)
    ↓ [unresolved or cross-domain]
Domain Owner reviews and decides (target: 10 business days)
    ↓ [cross-domain or policy conflict]
Governance Council resolves (next scheduled meeting)
    ↓ [requires resource or budget]
Executive Sponsor / CDO decision
```

### Steward Onboarding Checklist

- [ ] Role and responsibilities documented and signed off
- [ ] Time commitment acknowledged by steward and their manager
- [ ] Governance tools access provisioned (catalog, quality dashboard)
- [ ] Data domain scope briefing completed
- [ ] CDE inventory reviewed with outgoing steward or domain owner
- [ ] First 90-day goals set with domain owner
- [ ] Added to governance council communication distribution
- [ ] Paired with peer steward for first 30 days

---

## 4. Data Domain Management

### Domain Identification Methodology

A data domain is a bounded collection of data under unified business ownership. Domains are NOT org chart boxes - they represent logical subject areas that span systems.

**Step 1: Identify candidate domains from value streams**
Map major business value streams (Order-to-Cash, Source-to-Pay, Hire-to-Retire) and identify the core data entities each value stream produces or consumes.

**Step 2: Group by ownership, not by system**
Each domain should have a single accountable business owner. If two business areas both claim a dataset, it needs a domain boundary decision.

**Step 3: Validate domain boundaries**
- Each domain has a clear owner (one person or role)
- Domains do not significantly overlap in scope
- Each domain has at least 5-10 meaningful data entities
- Cross-domain data sharing points are documented

**Step 4: Map domains to source systems**
Identify which systems are the system of record for data in each domain.

### Domain-to-Function Mapping (Manufacturing Example)

| Data Domain | Business Owner | Systems of Record | Key Entities | Cross-Domain Dependencies |
|-------------|---------------|------------------|--------------|--------------------------|
| Product | VP Product Mgmt | PLM, ERP | Product, SKU, BOM, Specification | Customer, Supplier |
| Customer | VP Sales | CRM, ERP | Customer, Account, Contact, Contract | Order, Product |
| Supplier | VP Procurement | ERP, Supplier Portal | Supplier, Contract, Certificate | Product, Order |
| Order | VP Operations | ERP, OMS | Order, Line Item, Shipment | Customer, Product, Supplier |
| Finance | CFO | ERP, GL | GL Account, Cost Center, Budget, Invoice | Order, Supplier |
| Employee | CHRO | HCM, Active Directory | Employee, Role, Organization Unit | Customer, Finance |
| Equipment | VP Operations | IoT Platform, CMMS | Asset, Maintenance Record, Sensor | Product, Supplier |

### CDE Identification Methodology

Critical Data Elements are the subset of data elements whose accuracy, completeness, and consistency are essential to core business operations, reporting, or regulatory compliance.

**Step 1: Identify candidate CDEs per domain**
For each domain, list data elements that appear in: regulatory reports, executive dashboards, cross-system integrations, SLA/contractual obligations.

**Step 2: Score each candidate**

| Criterion | Score 1 (Low) | Score 2 (Medium) | Score 3 (High) |
|-----------|--------------|-----------------|----------------|
| Regulatory impact | No regulatory use | Referenced in reports | Required for compliance filing |
| Operational impact | Used in single process | Used in multiple processes | Blocks key operations if wrong |
| Reporting impact | Not in dashboards | In operational reports | In executive/board reporting |
| Cross-system use | Single system | 2-3 systems | 4+ systems / golden record |
| Data quality risk | Rarely wrong | Sometimes wrong | Frequently wrong or missing |

**Step 3: Prioritize by total score**
- Score 12-15: Tier 1 CDE (highest priority - address in Phase 1)
- Score 8-11: Tier 2 CDE (important - address in Phase 2)
- Score 5-7: Tier 3 CDE (monitor - address as capacity allows)
- Score <5: Not a CDE

**Target ratio:** CDEs should represent 5-15% of all data elements. If you are identifying more than 20%, you are not being rigorous enough.

### CDE Definition Template

```yaml
cde:
  name: "Customer Account Number"
  domain: "Customer"
  tier: 1
  definition: |
    The unique alphanumeric identifier assigned to each customer account
    in the CRM system. Used as the primary key for all customer-related
    transactions across ERP, CRM, and billing systems.
  business_owner: "VP Sales"
  data_steward: "Jane Smith, CRM Data Steward"
  system_of_record: "Salesforce CRM"
  consuming_systems:
    - SAP ERP (customer master)
    - Billing System
    - Customer Portal
  quality_rules:
    - "Not null for all active accounts"
    - "Format: ACC-XXXXXXXX (3 letters, dash, 8 digits)"
    - "Unique across all customer records"
    - "Present in SAP within 24 hours of CRM creation"
  regulatory_relevance: "Required in revenue reporting (SOX)"
  last_reviewed: "2026-01-15"
  review_cycle: "Annual"
  status: "Approved"
```

---

## 5. Maturity Assessment

### CMMI-Based 5-Level Model

| Level | Name | Description | Key Characteristics |
|-------|------|-------------|---------------------|
| **1** | Initial / Ad Hoc | Unpredictable, reactive | No defined processes, hero-dependent, inconsistent outcomes |
| **2** | Managed / Repeatable | Basic project-level processes | Some documented practices, inconsistent across org, silo-based |
| **3** | Defined | Standardized, proactive | Documented standards, enterprise-wide adoption, governance structure in place |
| **4** | Quantitatively Managed | Measured, controlled | KPIs tracked, quality measured, decisions data-driven |
| **5** | Optimizing / Continuous Improvement | Innovation-focused | Automation, ML-driven quality, self-service governance |

**Realistic targets for a 24-month program:**
- Current state for most enterprises starting governance: Level 1-2
- 12-month target: Level 2-3 in priority areas
- 24-month target: Level 3 in core areas, Level 2 in secondary areas
- Level 4/5: 3-5 year horizon

### Assessment by DMBOK Area

Use this worksheet format to baseline current state and define targets:

| DMBOK Area | Current Level | Evidence | Target (12 mo) | Target (24 mo) | Key Gap |
|------------|--------------|----------|----------------|----------------|---------|
| Data Governance | 1 | No formal roles or policies | 2 | 3 | Define operating model, assign stewards |
| Metadata Management | 1 | No catalog, informal glossary | 2 | 3 | Deploy catalog, publish 100 approved terms |
| Data Quality | 1 | No rules, reactive firefighting | 2 | 3 | Define DQ rules for Tier 1 CDEs |
| Reference & Master Data | 2 | Some MDM for customer, no governance | 3 | 3 | Govern existing MDM, expand to product |
| Data Security | 2 | Basic access controls, no classification | 3 | 3 | Implement 4-level classification scheme |
| Data Architecture | 1 | No standards, tribal knowledge | 2 | 3 | Document architecture standards |
| Data Integration | 2 | ETL pipelines exist, no standards | 2 | 3 | Define integration standards and lineage |

### Maturity Roadmap Pattern

**Phase 1 (Months 1-6): Foundation - Target Level 2**
- Establish governance operating model and council
- Assign domain owners for top 3 priority domains
- Identify and define Tier 1 CDEs (top 20-30 elements)
- Implement data classification scheme
- Deploy initial business glossary (50-100 approved terms)

**Phase 2 (Months 6-12): Structure - Target Level 3 in Priority Areas**
- Embed data stewards in all priority domains
- Complete CDE definitions for all Tier 1 CDEs
- Implement quality rules for Tier 1 CDEs
- Launch data quality dashboard for priority domains
- Achieve catalog coverage for priority systems

**Phase 3 (Months 12-24): Scale - Target Level 3 Broadly**
- Extend governance to all domains
- Automate quality monitoring and alerting
- Establish governance metrics and reporting to executives
- Launch data consumer self-service for glossary and lineage
- Achieve measurable quality improvement (target: >95% CDE quality score)

---

## 6. Data Quality Framework Design

### Six DQ Dimensions Table

| Dimension | Definition | Example Failure | Example Rule |
|-----------|-----------|-----------------|--------------|
| **Completeness** | Required fields are populated | Customer email is null in 30% of records | Email not null for all active customers |
| **Accuracy** | Data reflects real-world truth | Order quantity is 1000 when actual is 100 | Cross-validate order qty against warehouse receipt |
| **Consistency** | Same data is consistent across systems | Customer address differs between CRM and ERP | CRM address = ERP address within 24 hours of update |
| **Timeliness** | Data is current enough for its intended use | Yesterday's inventory data used for real-time order decisions | Inventory updated within 1 hour of warehouse transaction |
| **Validity** | Data conforms to defined format, range, and domain | Phone number contains letters; negative age | Phone matches regex; age between 0 and 120 |
| **Uniqueness** | No unintended duplicates | Same customer in CRM 3 times with different IDs | No two records with same name + address + email |

### Quality Rule Design Pattern (YAML)

```yaml
quality_rule:
  id: "QR-CUST-001"
  name: "Customer Email Completeness"
  cde: "Customer Email Address"
  domain: "Customer"
  dimension: "Completeness"
  description: |
    All active customer accounts must have a valid email address.
    Email is required for order confirmations and account management.
  rule_logic: |
    SELECT COUNT(*) as failing_records
    FROM customer_master
    WHERE status = 'ACTIVE'
      AND (email IS NULL OR email = '')
  threshold:
    warning: 1%      # Alert if >1% of active customers missing email
    critical: 5%     # Escalate if >5% of active customers missing email
  owner: "Jane Smith, Customer Data Steward"
  remediation_owner: "CRM Team"
  remediation_sla: "5 business days for critical threshold breach"
  frequency: "Daily"
  last_updated: "2026-01-15"
```

### Proactive vs Reactive Quality Strategy

| Approach | When to Use | Mechanisms | Ownership |
|----------|-------------|------------|-----------|
| **Reactive (firefighting)** | Current state for most orgs | Incident tickets, ad-hoc queries, manual review | IT/Engineering |
| **Proactive Detection** | Phase 2 target | Scheduled quality rules, automated monitoring, threshold alerts | Data Stewards + IT |
| **Proactive Prevention** | Phase 3 target | Input validation at source, workflow gates, MDM enrichment | Business + IT co-owned |
| **Self-Healing** | Level 4-5 maturity | ML anomaly detection, automated remediation, auto-merge duplicates | Platform/Engineering |

### Quality Scorecard Pattern

| Domain | CDE Count | Tier 1 Score | Tier 2 Score | Overall Score | Trend | Owner |
|--------|-----------|-------------|-------------|---------------|-------|-------|
| Customer | 18 CDEs | 94% | 87% | 91% | Up 3% | Jane Smith |
| Order | 12 CDEs | 98% | 91% | 95% | Stable | Bob Jones |
| Product | 22 CDEs | 76% | 71% | 74% | Down 2% | TBD - steward vacancy |
| Finance | 15 CDEs | 99% | 95% | 97% | Up 1% | Carol Davis |

**Scorecard Target:** Tier 1 CDEs >= 95% quality score across all domains by Month 18.

---

## 7. Governance Metrics and KPIs

### Program Health Metrics Table

| Metric | Description | Target | Frequency | Owner |
|--------|-------------|--------|-----------|-------|
| **CDE Quality Score** | % of Tier 1 CDEs meeting all quality rules | >95% | Weekly | Data Stewards |
| **Business Glossary Coverage** | % of data elements with approved business definitions | >80% of CDEs | Monthly | Governance Office |
| **Steward Coverage** | % of domains with active, named stewards | 100% of priority domains | Monthly | CDO |
| **Issue Resolution Rate** | % of data issues resolved within SLA | >90% within SLA | Monthly | Domain Owners |
| **Policy Compliance** | % of projects following governance processes | >85% | Quarterly | Governance Council |
| **Data Literacy Score** | % of data consumers completing governance training | >70% | Quarterly | Governance Office |
| **Governance Council Attendance** | % of council members attending monthly meeting | >80% | Monthly | CDO |
| **New CDE Definitions** | Number of new CDEs defined and approved per quarter | Roadmap-driven | Quarterly | Governance Office |
| **Quality Rule Coverage** | % of Tier 1 CDEs with at least one active quality rule | 100% by Month 12 | Monthly | Governance Office |

---

## 8. Data Classification

### Four-Level Classification Scheme

| Level | Label | Definition | Examples | Required Controls |
|-------|-------|-----------|---------|-------------------|
| **1** | Public | Approved for external publication | Product catalog, press releases, public pricing | None beyond standard publishing approval |
| **2** | Internal | For authorized employees and contractors only | Internal reports, org charts, process docs | Access requires employee authentication |
| **3** | Confidential | Sensitive business information; limited distribution | Customer contracts, financial forecasts, M&A data | Need-to-know access; encryption at rest |
| **4** | Restricted | Highly sensitive; regulatory or legal restrictions | PII, PHI, payment card data, trade secrets | Strict access controls, encryption, audit logging, regulatory compliance |

### PII / PHI Classification Table

| Data Element | Classification | Regulatory Driver | Masking Required |
|-------------|---------------|------------------|-----------------|
| Full Name | Restricted | GDPR, CCPA | Partial (first initial + last) in non-prod |
| Email Address | Restricted | GDPR, CCPA | Masked in non-prod |
| Social Security Number | Restricted | GDPR, IRS, SOX | Always masked; tokenize for processing |
| Date of Birth | Confidential | GDPR, HIPAA | Masked in non-prod |
| Home Address | Restricted | GDPR, CCPA | Masked in non-prod |
| Medical Diagnosis | Restricted | HIPAA | Always masked; encrypt at rest and transit |
| Payment Card Number | Restricted | PCI-DSS | Always tokenized; never store raw |
| IP Address | Confidential | GDPR | Anonymized in analytics |
| Employee ID | Internal | N/A | No masking required |
| Customer ID | Internal | N/A | No masking required |

### Sensitivity Tagging in Catalog

Every data asset in the catalog should have:
- Classification level (1-4 above)
- Sensitivity tags (PII, PHI, PCI, Financial, Confidential IP)
- Data residency requirement (if applicable)
- Retention period and disposal method
- Owner responsible for maintaining classification

---

## 9. Standards and Naming Conventions

### Naming Framework

| Object Type | Convention | Example |
|-------------|-----------|---------|
| Database / Schema | snake_case, domain prefix | `cust_master`, `ord_transactions` |
| Table | snake_case, noun plural | `customer_accounts`, `order_line_items` |
| Column | snake_case, descriptive | `customer_account_number`, `order_create_date` |
| Business Term | Title Case, no abbreviations | "Customer Account Number" not "CUST_ACCT_NUM" |
| Data Domain | Title Case, bounded noun | "Customer Domain", "Order Domain" |
| CDE ID | PREFIX-DOMAIN-SEQ | `CDE-CUST-001`, `CDE-ORD-015` |
| Quality Rule ID | QR-DOMAIN-SEQ | `QR-CUST-001`, `QR-FIN-022` |

### Business Term Definition Standard Template

```yaml
business_term:
  term: "Customer Account Number"
  definition: |
    The unique alphanumeric identifier assigned to each customer account
    at the time of account creation. This is the primary identifier used
    across all customer-facing and operational systems.
  domain: "Customer"
  synonyms:
    - "Account Number"
    - "Cust Acct Num"
    - "CUST_ID"
  related_terms:
    - "Customer Master Record"
    - "Account Status"
  example_values:
    - "ACC-00123456"
    - "ACC-00987654"
  steward: "Jane Smith"
  owner: "VP Sales"
  status: "Approved"  # Draft | Candidate | Approved | Deprecated
  approved_date: "2026-01-15"
  notes: |
    Prior to 2024, some legacy systems used a 6-digit numeric format.
    New format is ACC- followed by 8 digits. Legacy records have been migrated.
```

### Status Lifecycle

```
Draft → Candidate → Approved → Deprecated
  ↑          ↑                      ↓
  └──────────┴──────── Rejected ────┘
```

- **Draft**: Steward is authoring; not yet reviewed
- **Candidate**: Submitted for review; governance council will vote
- **Approved**: Ratified; published in catalog; authoritative definition
- **Deprecated**: No longer in use; replacement term linked
- **Rejected**: Failed review; returned to steward with feedback

---

## 10. Anti-Patterns

### Ten Common Governance Mistakes

| Anti-Pattern | Why It Fails | The Fix |
|-------------|-------------|---------|
| **Tool-first governance** | Buying Collibra before defining operating model, roles, or CDEs | Define operating model, roles, and top 20 CDEs before any tool selection |
| **IT-owned governance** | IT defines terms, assigns stewards, creates policies without business input | Business owns governance; IT facilitates. Stewards must be from the business. |
| **Boiling the ocean** | Attempting to catalog all 10,000 data elements from day one | Start with 20-30 Tier 1 CDEs. Prove value before expanding. |
| **Governance council with no authority** | Council meets monthly but cannot make binding decisions or require resources | Define and publish decision rights before first council meeting. Get executive mandate. |
| **Stewards with no time** | Naming someone a steward without adjusting their workload | Stewardship requires 15-25% dedicated time. Manager must acknowledge and protect it. |
| **One-size RACI** | Using the same RACI for a startup and a 10,000-person enterprise | Tailor RACI to organization size, domain complexity, and regulatory requirements. |
| **No CDE prioritization** | Treating all 500 identified data elements as equally important | Apply CDE scoring matrix. Tier 1 CDEs only in Phase 1. Ruthlessly prioritize. |
| **Governance without enforcement** | Publishing standards that projects are free to ignore | Integrate governance into project lifecycle. No data project goes live without steward sign-off. |
| **Over-engineering the operating model** | Creating 12 governance roles and 4 committees before a single CDE is defined | Start with 3 roles (Domain Owner, Steward, Custodian) and 1 committee. Add complexity when needed. |
| **Skipping data literacy** | Launching governance without educating consumers on how to use it | Build data literacy program alongside governance. Consumers need to understand why governance matters. |

---

## Quick Reference

### Governance Program Checklist (Program Launch)

- [ ] Executive sponsorship confirmed (CDO or equivalent)
- [ ] Operating model selected and documented
- [ ] Governance council chartered with decision rights
- [ ] Top 3 priority domains identified
- [ ] Domain owners assigned and confirmed
- [ ] Stewards nominated for priority domains
- [ ] 20-30 Tier 1 CDEs identified and scored
- [ ] CDE definitions drafted for Tier 1 CDEs
- [ ] Data classification scheme defined and approved
- [ ] Business glossary initial terms published (50+)
- [ ] Governance RACI signed off by council
- [ ] Governance metrics dashboard defined
- [ ] Tool selection criteria based on operating model requirements

### Governance Operating Model Decision Tree

```
Does the organization have a CDO or Chief Data Officer?
├── No → Start with Federated; plan for Hybrid within 18 months
└── Yes → Does the CDO have decision authority over data policies?
    ├── No (advisory only) → Federated with light coordination
    └── Yes → Does the org have strong autonomous business units?
        ├── Yes (multiple BUs with own P&L) → Hybrid
        └── No (single business, centralized) → Centralized or Hybrid
```

### DMBOK Area Quick Priority Guide

| Priority | Area | Why |
|----------|------|-----|
| Start here | Data Governance, Data Security | Foundation; everything depends on roles and classification |
| Phase 2 | Metadata Management, Reference & Master Data | Visibility and MDM governance |
| Phase 3 | Data Quality, Data Integration | Measurement and consistency |
| Phase 4+ | Data Architecture, Data Modeling, DW & BI | Standards and model governance |
| Ongoing | Document & Content Management | Often underinvested; tackle when foundation is stable |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Core skill content | ~/ali-ai/skills/ali-data-governance/SKILL.md |
| Data governance agent | ~/ali-ai/agents/ali-data-governance-expert.md |
| Collibra skill (tool-level) | ~/ali-ai/skills/ali-collibra/SKILL.md |
| Collibra expert agent | ~/ali-ai/agents/ali-collibra-expert.md |

---

**Document Version:** 1.0
**Last Updated:** 2026-02-18
**Maintained By:** ALI AI Team
