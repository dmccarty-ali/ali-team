---
name: ali-mdm-strategy
description: |
  Master Data Management strategy and architecture. Use when:

  PLANNING: Selecting MDM architecture style (Registry/Consolidation/Coexistence/Hub),
  designing golden record composition, planning survivorship rules, architecting
  multi-ERP consolidation strategy, scoping MDM data domains, phasing MDM roadmap,
  evaluating match-and-merge approaches, designing entity resolution at scale

  IMPLEMENTATION: Defining survivorship matrices, building source trust scoring,
  writing match key hierarchies, specifying identifier crosswalks, documenting
  canonical data models, creating code mapping tables across heterogeneous ERPs,
  designing steward review workflows for merge decisions

  GUIDANCE: Asking about MDM architecture trade-offs, golden record patterns,
  survivorship rule design, multi-ERP identifier management, MDM rollout sequencing,
  governance-to-MDM integration, entity resolution thresholds and false positive management

  REVIEW: Auditing MDM program design, validating architecture style justification,
  reviewing golden record attribute assignments, checking survivorship rule completeness,
  verifying match strategy thresholds, assessing roadmap phasing and realism

  DO NOT use for:
  - SAP MDG platform configuration (use ali-sap-mdg-expert)
  - BRFplus rules or SAP transaction codes (use ali-sap-mdg-expert)
  - Collibra governance platform setup (use ali-collibra)
  - Generic data modeling or warehouse design (use ali-data-architecture)
---

# MDM Strategy and Architecture

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Selecting MDM architecture style for an enterprise program
- Designing golden record composition across multiple source systems
- Planning survivorship rules and source trust scoring frameworks
- Architecting multi-ERP data consolidation (SAP, IFS, BAAN, Oracle, etc.)
- Scoping MDM data domains and prioritization
- Designing match-and-merge strategy and entity resolution approach
- Phasing an MDM roadmap (crawl/walk/run)

**Implementation:**
- Building survivorship matrices with attribute-level source priority
- Defining identifier crosswalks and code mapping tables
- Writing canonical data model definitions
- Designing steward review workflows for disputed merges
- Specifying blocking keys and scoring thresholds for matching

**Guidance/Best Practices:**
- Asking about MDM architecture style trade-offs
- Asking how to design survivorship rules for conflicting data sources
- Asking about golden record patterns for specific entities
- Asking about multi-ERP identifier management approaches
- Asking about governance-to-MDM integration patterns

**Review/Validation:**
- Auditing MDM program design decisions
- Reviewing golden record attribute assignments for completeness
- Checking survivorship rule quality and edge case coverage
- Validating match strategy thresholds and false positive risk
- Assessing MDM roadmap phasing and timeline realism

---

## Key Principles

- **Architecture style is a design constraint, not a product feature**: Choose style (Registry/Consolidation/Coexistence/Hub) based on system landscape and ownership model, not tool capabilities
- **Golden records are attribute-level, not system-level**: Different fields of the same entity may be owned by different source systems
- **Survivorship rules must handle every conflict case**: Missing rules produce unpredictable golden records
- **Match threshold calibration is iterative**: Auto-merge, review, and reject thresholds require tuning against actual data
- **Identifier crosswalks are the integration backbone**: Every source system must have a local-ID-to-MDM-ID mapping table
- **Domain sequencing matters**: Start with highest-value, lowest-complexity domains to build organizational trust
- **Governance drives MDM rules**: Data policies should define survivorship logic, not the other way around
- **Crawl/Walk/Run is not optional**: Enterprise MDM that skips phasing fails from scope overload

---

## 1. MDM Architecture Styles

### Four Architecture Styles

| Style | Description | Master Record Location | Source Systems Write To | Best For |
|-------|-------------|----------------------|------------------------|---------|
| **Registry** | MDM hub stores cross-references only; source systems remain systems of record | Source systems (hub has pointers) | Their own systems only | Minimal disruption; read-heavy golden record needs |
| **Consolidation** | Hub pulls data from sources, creates golden record for downstream consumption | MDM hub (read-only view) | Their own systems only | Analytics, reporting, BI; no writeback needed |
| **Coexistence** | Hub creates golden record AND publishes back to source systems | Both hub and sources | Source systems (hub reconciles) | Operational consistency needed across ERPs |
| **Transaction Hub** | Hub becomes system of record; source systems become transaction entry points | MDM hub exclusively | Via hub (hub distributes) | New implementations; greenfield; maximum control |

### Decision Matrix

| Factor | Registry | Consolidation | Coexistence | Transaction Hub |
|--------|----------|---------------|-------------|-----------------|
| Source system disruption | Low | Low | Medium | High |
| Writeback complexity | None | None | High | Very High |
| Operational consistency | None | None | High | Very High |
| Implementation risk | Low | Low | Medium | High |
| Time to value | Fast | Fast | Slow | Very Slow |
| Data quality enforcement | Weak | Medium | Strong | Strongest |
| Typical enterprise MDM | Common | Most common | Common | Rare |

### Gartner Alignment

Gartner classifies MDM styles on two axes: **operational vs analytical** and **hub-centric vs federated**. Mapping:

- **Registry** = Federated, analytical or operational
- **Consolidation** = Hub-centric, analytical
- **Coexistence** = Federated, operational
- **Transaction Hub** = Hub-centric, operational

### Hybrid Patterns

Most enterprise MDM programs use hybrid styles by domain:

| Domain | Style | Rationale |
|--------|-------|-----------|
| Customer Master | Coexistence | Operational systems need consistent customer IDs |
| Product/Material Master | Consolidation | Analytics and catalog use cases; ERPs own product creation |
| Vendor Master | Registry | Procurement owns vendors; MDM provides cross-reference only |
| GL Account | Transaction Hub | Finance consolidation requires hub as system of record |

### Re-Architecture Triggers

Consider migrating to a stronger style when:
- Source systems are being replaced or consolidated (ERP transformation)
- Data quality enforcement via Registry/Consolidation is insufficient
- Operational systems require real-time consistent master data
- MDM program reaches organizational maturity level 3+

---

## 2. Golden Record Design

### Attribute-Level Composition

Golden records are built field by field, not system by system. Each attribute has an independent source priority chain.

**Concept:** For a Customer entity, the legal name may be owned by SAP (billing system of record) while the preferred name is owned by CRM. Address is owned by the most recently validated source. Phone is owned by whichever system has the most recent update.

### Field-Level Source Priority Template

```
Entity: Customer Master
Attribute: Legal Name
  Priority 1: SAP (billing SOR) - use when populated
  Priority 2: IFS (ERP fallback) - use when SAP blank
  Priority 3: CRM (last resort) - use when both blank
  Null handling: FLAG as requires steward review

Attribute: Preferred Name / DBA
  Priority 1: CRM (customer-facing SOR)
  Priority 2: SAP (billing name truncation fallback)
  Null handling: Inherit from Legal Name

Attribute: Primary Address
  Priority 1: Most recently address-validated source (any system)
  Priority 2: SAP (billing address)
  Priority 3: IFS
  Null handling: FLAG for steward review

Attribute: Tax ID (EIN/SSN)
  Priority 1: SAP (finance SOR)
  Null handling: BLOCK golden record publication until populated
```

### Conflict Resolution Patterns

| Conflict Type | Resolution Approach |
|---------------|---------------------|
| Two systems have different values, both non-null | Apply field-level trust score; higher score wins |
| One system null, one populated | Use populated value regardless of trust score |
| All systems null | Apply null handling rule (flag, block, inherit, default) |
| Format conflict (e.g., "ST" vs "Street") | Normalize to canonical format before comparison |
| Temporal conflict (different effective dates) | Use most recent effective date; preserve history |
| Both systems equally trusted | Escalate to steward review queue |

### Composite Golden Record Patterns

When entities span multiple source systems with non-overlapping attributes:

**Pattern: Additive Composition**
SAP owns financial attributes. IFS owns manufacturing attributes. CRM owns contact attributes. Golden record = union of all non-conflicting attributes, with survivorship applied only where overlap exists.

**Pattern: Hierarchical Master**
Parent record in MDM hub. Child attributes sourced independently per system. Parent ID is the golden identifier; child attributes are versioned per source.

---

## 3. Survivorship Rules

### Source Trust Scoring

Trust scores are assigned per source system per data domain. They are inputs to survivorship logic, not the survivorship rule itself.

**Dimensions of trust:**

| Dimension | Description | Measurement |
|-----------|-------------|-------------|
| **Recency** | How current is the data? | Days since last update |
| **Frequency** | How often is it updated? | Updates per month |
| **Accuracy** | How often is it correct? | Validation pass rate % |
| **Completeness** | How complete are required fields? | % non-null on required attributes |
| **Authority** | Is this system designated SOR for this attribute? | Binary: yes/no |

**Trust score formula (example):**
```
Trust = (Authority_weight × Authority) +
        (Accuracy_weight × Accuracy%) +
        (Completeness_weight × Completeness%) +
        (Recency_weight × Recency_score)

Weights typically: Authority=40%, Accuracy=25%, Completeness=20%, Recency=15%
```

### Survivorship Matrix Template

| Attribute | SAP Trust | IFS Trust | BAAN Trust | CRM Trust | Rule | Null Handling |
|-----------|-----------|-----------|------------|-----------|------|---------------|
| Legal Name | 0.90 | 0.70 | 0.60 | 0.50 | Highest trust wins | FLAG |
| Tax ID | 0.95 | 0.40 | 0.40 | N/A | SAP always wins | BLOCK |
| Primary Address | Dynamic | Dynamic | Dynamic | Dynamic | Most recently validated | FLAG |
| Phone | 0.50 | 0.60 | 0.50 | 0.80 | CRM wins (customer-facing) | Inherit contact |
| Credit Limit | 0.95 | N/A | N/A | N/A | SAP always wins | Default=0 |
| Preferred Name | 0.60 | N/A | N/A | 0.90 | CRM wins | Inherit Legal Name |

### Null Handling Rules

| Rule | Behavior | When to Use |
|------|----------|-------------|
| **FLAG** | Populate golden record but mark attribute as unverified | Non-blocking but needs attention |
| **BLOCK** | Do not publish golden record until resolved | Critical identifiers (Tax ID, primary key) |
| **INHERIT** | Copy value from related attribute | Secondary names, derived fields |
| **DEFAULT** | Use predefined default value | Numeric fields where 0 is valid default |
| **SKIP** | Leave blank in golden record | Truly optional attributes |

### Format Conflict Resolution

Common format conflicts requiring normalization before survivorship:

| Conflict Type | Example | Resolution |
|---------------|---------|-----------|
| Name casing | "ACME CORP" vs "Acme Corp" | Normalize to title case before compare |
| Address abbreviation | "Street" vs "St" vs "ST" | Normalize via USPS/postal authority lookup |
| Phone format | "5551234567" vs "(555) 123-4567" | Strip to E.164 format before compare |
| Country codes | "US" vs "USA" vs "United States" | Map to ISO 3166-1 alpha-2 |
| Date format | "01/15/2024" vs "2024-01-15" | Normalize to ISO 8601 |
| Boolean | "Y/N" vs "1/0" vs "true/false" | Normalize to boolean before compare |

---

## 4. Match & Merge Strategy

### Matching Approaches

| Approach | How It Works | Precision | Recall | Use Case |
|----------|-------------|-----------|--------|---------|
| **Deterministic** | Exact match on defined keys (SSN, Tax ID, email) | Very High | Low-Medium | When authoritative identifiers exist |
| **Probabilistic (Fellegi-Sunter)** | Weighted scoring across multiple attributes | High | High | Customer/vendor matching without clean IDs |
| **ML-based** | Trained classifier on labeled match/non-match pairs | Very High | Very High | Large scale; when training data available |
| **Hybrid** | Deterministic first pass, probabilistic for remainder | Very High | Very High | Most enterprise MDM programs |

### Blocking Keys

Blocking reduces the candidate pair space from O(n²) to manageable size. Without blocking, matching 10M records requires 50 trillion comparisons.

**Common blocking strategies:**

| Entity | Blocking Key Options | Notes |
|--------|---------------------|-------|
| Customer | Postal code + first 3 chars of name | Effective for B2C |
| Customer | Tax ID prefix + state | Effective for B2B |
| Vendor | DUNS number prefix | If DUNS available |
| Vendor | Tax ID + country code | Most reliable |
| Product | Category code + manufacturer code | For manufactured goods |
| Product | UPC prefix + brand code | For retail products |
| Employee | First 3 chars of last name + birth year | For HR master |

**Multi-pass blocking:** Run multiple blocking passes and union the candidate sets. Increases recall at cost of compute.

### Scoring Thresholds

| Zone | Score Range | Action | Review Queue |
|------|-------------|--------|-------------|
| **Auto-merge** | 0.95 - 1.00 | Merge automatically | None |
| **Review** | 0.75 - 0.94 | Send to steward review | Steward queue |
| **Potential match** | 0.50 - 0.74 | Log for batch review | Batch review queue |
| **Reject** | 0.00 - 0.49 | Do not merge | None |

Thresholds are calibrated against your data. Initial thresholds should be conservative (auto-merge only at 0.95+). Tune down as false positive rate is measured.

### False Positive and False Negative Management

**False positive** (merging distinct entities): Higher cost. Causes data corruption, incorrect billing, regulatory risk.

**False negative** (missing true duplicates): Lower cost. Leaves duplicates in system; cleaner data but incomplete golden record.

**Calibration approach:**
1. Sample 500+ record pairs from the review zone
2. Have stewards manually classify each as match/non-match
3. Measure precision and recall at each threshold
4. Adjust thresholds to balance acceptable false positive rate vs recall
5. Re-calibrate quarterly as data volumes and patterns change

### Match Key Hierarchy

```
Level 1 (Deterministic - exact):
  Tax ID match → Auto-merge (score = 1.0)
  DUNS Number match → Auto-merge (score = 1.0)
  Email match (normalized) → Score = 0.80 (partial confidence)

Level 2 (Probabilistic - weighted):
  Legal Name (normalized) → Weight = 0.30
  Primary Address (normalized) → Weight = 0.25
  Phone (normalized) → Weight = 0.20
  ZIP Code → Weight = 0.10
  Industry Code → Weight = 0.15

Composite score = L1 contribution + (L2 weights × L2 similarities)
```

---

## 5. Multi-ERP Consolidation

### Heterogeneous ERP Landscape

Most enterprise MDM programs must consolidate master data across at least two of:

| ERP | Common Use | Master Data Strengths | Common Challenges |
|-----|-----------|----------------------|-------------------|
| **SAP** | Finance, procurement, manufacturing | Vendor master, GL account, customer (FI/SD) | Complex client/company code hierarchy; BRFplus rules |
| **IFS** | Manufacturing, service, asset-intensive industries | Work orders, product/material, service contracts | Non-standard customer model; limited MDM tooling |
| **BAAN/Infor LN** | Manufacturing, discrete industries | Item master, BOM, routing | Legacy data quality issues; non-standard identifiers |
| **Oracle EBS/Cloud** | Finance, HR, procurement | Supplier master, item master | Multi-org complexity; frequent release changes |
| **Microsoft D365** | SMB to mid-market | Customer, vendor, product | Customization sprawl; limited lineage |

### Semantic Mapping Challenges

Same business concept, different data models across ERPs:

| Business Concept | SAP | IFS | BAAN |
|-----------------|-----|-----|------|
| Customer identifier | Customer (KUNNR) | Customer ID | Customer Code |
| Vendor identifier | Vendor (LIFNR) | Supplier ID | Supplier Code |
| Legal entity | Company Code | Company | Financial Company |
| Product unit | Base Unit of Measure | Unit | Stock Unit |
| Address type | Partner Function (SP/SH/BP) | Address Type Code | Address Role |
| Customer status | Account Group + Partner | Customer Category | Customer Class |

### Code Mapping Tables

Code mapping tables translate source-system-specific codes to canonical MDM codes and back.

**Pattern:**
```
Table: mdm_code_map
  domain          VARCHAR  -- e.g., 'customer_status'
  source_system   VARCHAR  -- e.g., 'SAP', 'IFS', 'BAAN'
  source_code     VARCHAR  -- e.g., 'D001' (SAP account group)
  canonical_code  VARCHAR  -- e.g., 'DISTRIBUTOR'
  canonical_label VARCHAR  -- e.g., 'Distributor'
  effective_from  DATE
  effective_to    DATE
  notes           TEXT
```

**Example - Customer Type mapping:**

| Domain | Source System | Source Code | Canonical Code | Canonical Label |
|--------|--------------|-------------|----------------|-----------------|
| customer_type | SAP | D001 | DISTRIBUTOR | Distributor |
| customer_type | SAP | Z001 | END_CUSTOMER | End Customer |
| customer_type | IFS | DIST | DISTRIBUTOR | Distributor |
| customer_type | IFS | RETAIL | END_CUSTOMER | End Customer |
| customer_type | BAAN | D | DISTRIBUTOR | Distributor |
| customer_type | BAAN | EC | END_CUSTOMER | End Customer |

### Identifier Crosswalk

Every source system record must be linked to its MDM golden record ID via a crosswalk table. This is the integration backbone.

```
Table: mdm_identifier_crosswalk
  mdm_id          VARCHAR  -- MDM hub golden record ID
  source_system   VARCHAR  -- e.g., 'SAP_ECC', 'IFS_PROD', 'BAAN_EU'
  source_id       VARCHAR  -- Source system's local identifier
  entity_domain   VARCHAR  -- e.g., 'CUSTOMER', 'VENDOR', 'PRODUCT'
  match_confidence DECIMAL -- 0.0-1.0; 1.0 = deterministic match
  created_at      TIMESTAMP
  last_verified   TIMESTAMP
  status          VARCHAR  -- ACTIVE, SUSPECT, INACTIVE
```

**Population strategy:**
1. Deterministic matches (Tax ID, DUNS): Populate with confidence = 1.0
2. Probabilistic matches above auto-merge threshold: Populate with match confidence score
3. Manual steward assignments: Populate with confidence = 1.0 (human-verified)
4. Unmatched records: Create new MDM ID, single-source crosswalk row

### Canonical Data Model

The canonical model is the entity definition that the MDM hub uses, independent of any source system's schema.

**Pattern for Customer canonical model:**
```
MDM_CUSTOMER (canonical)
  mdm_customer_id     -- Hub-generated GUID
  legal_name          -- From survivorship
  preferred_name      -- From survivorship
  customer_type       -- Canonical code from code map
  tax_id              -- From survivorship (SAP priority)
  primary_address     -- From survivorship (most recently validated)
  primary_phone       -- From survivorship
  primary_email       -- From survivorship
  credit_limit        -- From survivorship (SAP finance SOR)
  credit_currency     -- From survivorship (SAP finance SOR)
  status              -- From survivorship
  created_at          -- MDM hub timestamp
  last_updated        -- MDM hub timestamp
  golden_record_version -- Increments on any survivorship change
```

### Patterns by Entity

**Customer Master - Coexistence Pattern:**
- SAP owns billing customer (sold-to, ship-to, bill-to)
- CRM owns prospect-to-customer lifecycle
- MDM creates single customer ID that both systems reference
- Address validation service called at golden record creation
- Crosswalk: SAP KUNNR ↔ MDM_ID ↔ CRM Account ID

**Vendor Master - Consolidation Pattern:**
- SAP owns vendor payment data
- IFS owns vendor production/sourcing data
- MDM golden record used for procurement analytics and reporting
- No writeback to source systems
- Crosswalk: SAP LIFNR ↔ MDM_ID ↔ IFS Supplier ID

**Product/Material Master - Consolidation Pattern:**
- SAP MM owns finished goods and raw materials
- IFS owns manufactured components and BOM
- MDM golden record is read-only product catalog
- Item classification hierarchy normalized to canonical taxonomy
- Crosswalk: SAP MATNR ↔ MDM_ID ↔ IFS Part Number

**GL Account - Registry Pattern:**
- SAP owns chart of accounts (source of record)
- IFS has parallel account structure (local codes)
- MDM stores crosswalk and canonical account hierarchy
- No golden record merge; registry maps local codes to group codes
- Crosswalk: SAP account ↔ Group account code ↔ IFS account

---

## 6. MDM Data Domains

### Standard MDM Domains

| Domain | Entities | Typical Priority | Complexity |
|--------|---------|-----------------|-----------|
| **Customer** | Sold-to, ship-to, bill-to, contact | High | High (many source systems, high match complexity) |
| **Vendor / Supplier** | Vendor, supplier, contractor | High | Medium (cleaner identifiers, less writeback) |
| **Product / Material** | Item, SKU, part, material | High | High (taxonomy complexity, BOM hierarchy) |
| **Employee** | Worker, staff, contractor | Medium | Medium (HR privacy constraints) |
| **Location** | Facility, site, warehouse, plant | Medium | Low (fewer sources, stable data) |
| **GL Account** | Account, cost center, profit center | Medium | Low (finance SOR usually clear) |
| **Asset** | Equipment, machinery, infrastructure | Low | Medium (varies by industry) |
| **Hierarchy** | Customer hierarchy, org hierarchy | Low | High (parent-child complexity) |

### Domain Prioritization Framework

Score each domain across four factors (1=low, 5=high):

| Factor | Weight | Description |
|--------|--------|-------------|
| Business value | 40% | Revenue impact, operational dependency, regulatory risk |
| Data quality pain | 30% | Current duplicate rate, business complaints, audit findings |
| Feasibility | 20% | Number of source systems, data quality baseline, SME availability |
| Strategic alignment | 10% | Executive sponsorship, transformation program alignment |

**Priority score = (Business_value × 0.4) + (DQ_pain × 0.3) + (Feasibility × 0.2) + (Strategic × 0.1)**

### Phased Rollout Patterns

**Pattern A: Single Domain First (Recommended for new programs)**
- Phase 1: Vendor Master (cleaner data, less operational risk)
- Phase 2: Customer Master (higher complexity, build on Phase 1 lessons)
- Phase 3: Product Master (taxonomy work, requires Phase 2 governance patterns)
- Phase 4+: Remaining domains

**Pattern B: Single Business Unit First**
- Phase 1: All domains for one business unit or geography
- Phase 2: Expand to second business unit, reuse domain models
- Advantage: Organizational depth before breadth

**Pattern C: Regulatory-Driven**
- Phase 1: Domains required for regulatory compliance (e.g., customer for KYC/AML)
- Phase 2: Operational domains
- Advantage: Clear business justification for initial investment

---

## 7. MDM & Governance Integration

### Stewards as Merge Reviewers

Data stewards in the governance program are the human escalation path for MDM merge decisions:

| Trigger | Queue | Steward Role |
|---------|-------|-------------|
| Match score in review zone (0.75-0.94) | Merge review queue | Review pair, approve or reject merge |
| All sources null on required field | Data quality queue | Source correct value, update source system |
| Survivorship conflict with equal trust scores | Conflict resolution queue | Designate system of record for this instance |
| New record from source with no match | Validation queue | Confirm truly new entity vs missed duplicate |

### Governance Policies Driving MDM Rules

MDM survivorship and matching rules should be derived from governance policies, not invented independently:

| Governance Policy | MDM Rule Derived |
|------------------|-----------------|
| "SAP is system of record for all financial master data" | SAP trust score = 1.0 for financial attributes; all other sources = 0.0 |
| "Customer address must be USPS-validated within 90 days" | Address recency weight = 90-day decay function; USPS validation flag required |
| "No vendor can be merged without procurement approval" | Vendor merge requires procurement steward approval regardless of match score |
| "Duplicate customers are a P1 data quality issue" | Customer domain match thresholds are more aggressive; false negative rate tolerated lower |

### Collibra Integration Points

When Collibra is part of the governance stack:

| Integration | Purpose | Direction |
|------------|---------|-----------|
| Business glossary → MDM attribute definitions | Collibra terms define canonical attribute names and descriptions | Collibra → MDM hub |
| Data quality rules → Survivorship inputs | Quality scores from Collibra DQ scorecards feed source trust scores | Collibra DQ → MDM |
| MDM golden records → Collibra data catalog | Golden record schema published as certified data assets in Collibra | MDM hub → Collibra |
| Steward workflow → MDM merge review | Collibra workflow used as merge review UI | Collibra workflow → MDM API |
| MDM match decisions → Collibra lineage | Merge decisions logged as lineage events | MDM → Collibra lineage |

---

## 8. MDM Roadmap Patterns

### Crawl / Walk / Run

| Phase | Duration | Scope | Key Deliverables |
|-------|----------|-------|-----------------|
| **Crawl** | 3-6 months | Foundation | Architecture style selected; first domain scoped; tooling selected; governance integration designed |
| **Walk** | 6-12 months | First domain live | First golden record domain operational; steward workflows live; crosswalks populated; monitoring in place |
| **Run** | 12-24 months | Scale out | Additional domains added; coexistence writeback enabled; advanced matching (ML); self-service stewardship |

### Timeline Patterns for Enterprise MDM

**Small enterprise (2-3 ERP systems, 1-2 domains):**
- Crawl: 3 months
- Walk (first domain): 6 months
- Total to first production golden record: 9 months

**Mid-size enterprise (3-5 ERP systems, 3-4 domains):**
- Crawl: 4-6 months
- Walk (first domain): 9-12 months
- Run (additional domains): 6-9 months per domain
- Total to full scope: 24-36 months

**Large enterprise (5+ ERP systems, 5+ domains, global):**
- Crawl: 6-9 months
- Walk (first domain, one region): 12-18 months
- Run (global rollout per domain): 12-18 months per domain
- Total to full scope: 4-7 years (phased over transformation program)

### Quick Wins for Early Stakeholder Value

Deliver visible wins within the first 90 days of the program:

| Quick Win | Effort | Value | Audience |
|-----------|--------|-------|---------|
| Duplicate vendor report | Low | Medium | Procurement, Finance |
| Customer match preview (read-only) | Medium | High | Sales, CRM team |
| Cross-system identifier crosswalk report | Low | Medium | IT, Integration team |
| Survivorship rule documentation | Low | Medium | Business owners |
| Golden record design workshop output | Low | High | Executive sponsors |

---

## 9. Entity Resolution at Scale

### Blocking Strategies for Large Datasets

For datasets over 10M records, standard all-pairs matching is computationally infeasible.

**Sorted Neighborhood Method:**
Sort records by a blocking key (e.g., normalized name). Compare each record only to the next W records (sliding window). Effective for name-based matching.

**Canopy Clustering:**
Create loose clusters (canopies) using a cheap distance metric. Apply expensive matching only within canopies. Tunable via canopy thresholds T1 (join) and T2 (core).

**LSH (Locality-Sensitive Hashing):**
Hash records so similar records collide in the same hash bucket. Sublinear time blocking. Best for high-dimensional attribute similarity.

**Multi-pass blocking:**
Run 3-5 different blocking keys independently. Union the candidate sets. Increases recall at the cost of compute but remains tractable.

### Distributed Matching

For production-scale matching (100M+ records):

| Approach | Technology | Use Case |
|----------|-----------|---------|
| Spark-based matching | Dedupe.io, Zingg | Open-source, batch processing |
| Cloud MDM | Informatica MDM, IBM MDM | Enterprise platform with built-in matching |
| Streaming matching | Kafka + ML model | Real-time new record matching |
| Graph-based | Neo4j entity resolution | Complex relationship traversal |

### Incremental Matching

After initial bulk matching, new records are matched incrementally:

1. New record arrives from source system
2. Blocking keys extracted from new record
3. Candidate pool retrieved from MDM hub using blocking keys
4. Match scores computed against candidates
5. If auto-merge: Create crosswalk entry, update golden record
6. If review: Route to steward queue
7. If no match: Create new MDM ID, single-source crosswalk
8. Golden record updated via survivorship recalculation

### Performance Considerations

| Optimization | Impact | When to Apply |
|-------------|--------|--------------|
| Pre-compute blocking indexes | Very High | Always |
| Normalize before storage (not at match time) | High | Always |
| Cache high-frequency lookup records | High | Vendor/product masters |
| Async merge for non-critical domains | Medium | Analytics-only golden records |
| Tiered match (cheap deterministic first) | High | Always |

---

## 10. Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|--------------|--------------|-----|
| **System-level survivorship** (SAP wins everything) | Over-reliance on one system ignores superior data in others; creates data quality debt | Define attribute-level source priority per field, not per system |
| **No blocking keys** | All-pairs matching is computationally infeasible above 1M records | Define at least 3 blocking strategies before matching any production data |
| **Single trust score per system** | Different systems have different quality on different attributes | Score trust at attribute-domain level, not system level |
| **Missing code mapping tables** | Canonical codes become meaningless without reverse-mapping to source codes | Build bidirectional code maps before integration goes live |
| **Identifier crosswalk as afterthought** | Integration breaks when source IDs change or new systems are added | Design crosswalk table as core architecture artifact, not integration utility |
| **Starting with customer domain** | Highest complexity, most political, highest failure risk for program's first domain | Start with vendor master or location; lower complexity, faster win |
| **Auto-merge threshold too low** | Merging distinct entities corrupts golden record; business trust collapses | Start conservative (0.95+); tune down only after measuring false positives |
| **No steward workflow for review zone** | Review-zone matches sit unresolved; duplicates persist indefinitely | Build steward review queue as part of Phase 1; not optional |
| **Governance and MDM as separate programs** | Duplicate stewardship efforts; conflicting policies | Design jointly from the start; governance policies should derive MDM rules |
| **Big bang all domains** | Scope overload; no quick wins; program canceled before delivery | Phase by domain; deliver first golden record within 9-12 months |

---

## Quick Reference

### Architecture Style Selection

```
Q1: Do source systems need to receive golden record updates?
  No  → Consolidation or Registry
  Yes → Coexistence or Transaction Hub

Q2: Should MDM hub be system of record?
  No  → Registry or Consolidation
  Yes → Transaction Hub

Q3: Is analytical use case primary?
  Yes → Consolidation (analytics golden record)
  No  → Coexistence (operational consistency)

Q4: Minimize source system disruption?
  Yes → Registry (pointer-only hub)
  No  → evaluate Coexistence or Hub
```

### Survivorship Rule Checklist

- [ ] Source trust scores defined for every source system × data domain combination
- [ ] Field-level priority chain documented for every attribute
- [ ] Null handling rule specified for every attribute (FLAG / BLOCK / INHERIT / DEFAULT / SKIP)
- [ ] Format normalization rules defined before survivorship applied
- [ ] Conflict resolution rule defined for equal-trust-score scenarios
- [ ] Temporal handling rule defined for effective-date conflicts
- [ ] Survivorship matrix signed off by business domain owners

### Match Threshold Starting Points

| Domain | Auto-Merge Start | Review Zone | Notes |
|--------|-----------------|-------------|-------|
| Customer (B2B) | 0.95 | 0.75-0.94 | High false-positive cost |
| Customer (B2C) | 0.90 | 0.70-0.89 | Higher volume; more tuning |
| Vendor | 0.92 | 0.75-0.91 | Procurement approval gate anyway |
| Product / Material | 0.95 | 0.80-0.94 | BOM integrity risk |
| Employee | 0.98 | 0.90-0.97 | Privacy and HR sensitivity |

### Domain Phasing Quick Decision

**Phase 1 (Lowest Risk):** Vendor Master, Location, GL Account
**Phase 2 (Medium Risk):** Customer Master, Employee
**Phase 3 (Highest Risk / Complexity):** Product Master, Asset, Hierarchy

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| MDM strategy skill | ~/ali-ai/skills/ali-mdm-strategy/SKILL.md |
| MDM strategy agent | ~/ali-ai/agents/ali-mdm-strategy-expert.md |
| SAP MDG platform expert | ~/ali-ai/agents/ali-sap-mdg-expert.md |
| Collibra governance expert | ~/ali-ai/agents/ali-collibra-expert.md |
| Collibra skill | ~/ali-ai/skills/ali-collibra/SKILL.md |

---

**Document Version:** 1.0
**Last Updated:** 2026-02-18
**Maintained By:** ALI AI Team
