# ali-collibra - Implementation Patterns Reference

## Implementation Patterns

### Greenfield Implementation

```yaml
# Phase 1: Foundation (Weeks 1-4)
- Setup Collibra instance (cloud or on-prem)
- Define Operating Model hierarchy
  - Communities
  - Domains
  - Initial asset types
- Configure authentication (Azure AD SAML)
- Setup user roles and permissions

# Phase 2: Catalog & Glossary (Weeks 5-8)
- Deploy Collibra Edge (if on-prem sources)
- Configure catalog scanners for key databases
- Import business glossary terms
- Define initial governance workflows
- Assign data stewards to domains

# Phase 3: Lineage & Quality (Weeks 9-12)
- Configure automated lineage scanners
- Define data quality rules
- Setup quality scorecards
- Configure alerting and notifications

# Phase 4: Adoption & Scale (Weeks 13-16)
- User training and enablement
- Expand to additional data sources
- Refine governance workflows based on feedback
- Implement data marketplace features
```

### Brownfield Migration

```yaml
# Migrating from legacy governance tools
migration_pattern:
  1. Inventory existing metadata
     - Extract from current tool (API or export)
     - Map to Collibra asset types

  2. Define mapping rules
     - Legacy term → Collibra Business Term
     - Legacy table → Collibra Technical Asset
     - Legacy lineage → Collibra Relations

  3. Build migration scripts
     - Python scripts using Collibra REST API
     - Batch processing with error handling
     - Validation and reconciliation

  4. Parallel run period
     - Run both systems for 30-60 days
     - Compare outputs for consistency
     - User acceptance testing

  5. Cutover
     - Final migration batch
     - Decommission legacy tool
     - Redirect all traffic to Collibra
```

### Phased Rollout by Domain

```yaml
# Rolling out Collibra domain by domain
phase_1: Finance Domain
  - Scope: Tax practice databases (Snowflake)
  - Users: 5 stewards, 20 consumers
  - Timeline: 4 weeks
  - Success criteria: 90% metadata coverage, 5 approved workflows

phase_2: Medical EDI Domain
  - Scope: Claims processing databases
  - Users: 8 stewards, 50 consumers
  - Timeline: 6 weeks
  - Success criteria: Automated lineage for 80% of pipelines

phase_3: Client Services Domain
  - Scope: CRM and client engagement data
  - Users: 10 stewards, 100 consumers
  - Timeline: 6 weeks
  - Success criteria: Data quality scorecards green for 85% of tables
```
