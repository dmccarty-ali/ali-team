# ali-sap-mdg - Implementation Details Reference

This reference contains detailed implementation patterns, timelines, and project structure guidance for SAP MDG implementations.

---

## Business Partner vs Legacy Migration

**Business Partner (BP) is mandatory in S/4HANA:**
- Replaces KNA1/KNB1 (customer master)
- Replaces LFA1/LFB1 (vendor master)
- Unified object for all external parties
- Extensible for custom roles (e.g., prospects, partners)

**Migration Pattern:**
```
S/4HANA conversion: KNA1 → BUT000 (one-time migration)
Ongoing MDG: Create/update Business Partner directly (no legacy tables)
```

---

## Domain Complexity and Timelines

| Domain | Complexity | Typical Timeline | Why Complex |
|--------|------------|------------------|-------------|
| Customer (BP) | Medium | 12-16 weeks | Duplicate detection, hierarchy management, multi-role |
| Material | High | 16-20 weeks | BOM structures, classification, lifecycle management |
| Supplier (BP) | Low-Medium | 10-12 weeks | Simpler than Customer, risk/compliance workflows |
| Finance | Medium | 12-14 weeks | Chart of accounts alignment, validation rules |
| Custom | Varies | 8-20 weeks | Depends on data model complexity |

---

## Implementation Patterns

### Greenfield (New MDG Installation)

**Typical 6-9 month timeline for 3 domains:**

**Phase 1: Foundation (8 weeks)**
- Install S/4HANA hub or activate MDG in co-deployment
- Configure system landscape (DEV/QA/PROD)
- Setup transport routes
- Define governance organization and roles

**Phase 2: Domain 1 - Customer (12-16 weeks)**
- Data model configuration (Business Partner)
- Matching rules and survivorship
- BRFplus workflow design
- DQM address validation setup
- Integration to target systems (MDI or IDoc)
- User acceptance testing (UAT)

**Phase 3: Domain 2 - Material (16-20 weeks)**
- Material master configuration
- BOM and classification setup
- Workflow for lifecycle management
- Integration to manufacturing systems
- UAT

**Phase 4: Domain 3 - Supplier (10-12 weeks)**
- Supplier master (Business Partner)
- Risk/compliance workflows
- Integration to procurement systems
- UAT

**Total: 6-9 months for 3 domains**

### Brownfield (Extending Existing MDG)

**Faster than greenfield (infrastructure exists):**

**Adding new domain: 3-6 months**
- Reuse existing hub infrastructure
- Reuse BRFplus patterns from other domains
- Focus on domain-specific data model and workflows

**Extending existing domain: 1-3 months**
- Add fields to existing data model
- Modify workflows (BRFplus decision tables)
- Update integrations

### Phased Domain Rollout

**Recommended order:**

1. **Start with Supplier** (10-12 weeks) - Simplest domain, quick win
2. **Then Customer** (12-16 weeks) - Higher complexity, high business value
3. **Then Material** (16-20 weeks) - Most complex, benefits from lessons learned

### Dual-ERP Scenarios

**When running SAP ECC and S/4HANA in parallel during migration:**

**Pattern 1: MDG Hub as Bridge**
```
ECC (legacy) ←→ MDG Hub ←→ S/4HANA (new)
                   ↓
              Golden Record
```

**Pattern 2: Staged Migration**
```
Phase 1: ECC → MDG (govern legacy data)
Phase 2: ECC + S/4HANA → MDG (dual source)
Phase 3: S/4HANA → MDG (legacy decommissioned)
```

---

## Typical Timelines

### By Domain (Including Design, Config, Testing, UAT)

| Domain | Greenfield | Brownfield (add to existing MDG) |
|--------|------------|----------------------------------|
| **Customer (BP)** | 12-16 weeks | 8-12 weeks |
| **Material** | 16-20 weeks | 10-14 weeks |
| **Supplier (BP)** | 10-12 weeks | 6-10 weeks |
| **Finance** | 12-14 weeks | 8-12 weeks |
| **Custom** | 8-20 weeks | 6-16 weeks |

### Full MDG Program

| Scenario | Timeline | Notes |
|----------|----------|-------|
| **Greenfield 3 domains** | 6-9 months | Sequential domain implementation |
| **Brownfield extension** | 3-6 months | Reuse existing infrastructure |
| **Single domain (simple)** | 3-5 months | Co-deployment, no matching/consolidation |
| **Single domain (complex)** | 4-7 months | Hub deployment with matching |

**Factors that extend timeline:**
- Complex matching rules (add 4-6 weeks)
- Custom domain data models (add 3-8 weeks)
- Integration to >5 target systems (add 2-4 weeks per system)
- Extensive DQM cleansing required (add 4-8 weeks)
- Organizational change management (add 2-4 weeks)

---

## Project Structure

**Note:** SAP MDG is a commercial product; we do not maintain MDG source code. Configuration and extensions are project-specific.

**Typical project structure:**
```
/config/
  mdg/
    brf_rules/          BRFplus decision table exports
    matching_rules/     Matching configuration (USMD_RULE exports)
    data_model/         Custom entity definitions (USMD_MODEL exports)
    workflows/          Workflow configuration documentation
    integrations/       MDI/IDoc configuration specs
/docs/
  mdg/
    architecture.md     Hub vs co-deployment decisions
    domain_design.md    Domain-specific data model and rules
    transport_plan.md   Transport route and cutover plan
/scripts/
  mdg/
    data_migration/     Data migration scripts (legacy to MDG)
    test_data/          Test data generation for UAT
```
