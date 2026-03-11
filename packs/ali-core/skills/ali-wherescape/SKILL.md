---
name: ali-wherescape
description: |
  WhereScape data warehouse automation platform expertise. Use when:

  PLANNING: Designing WhereScape data warehouse automation, architecting
  template-based code generation, planning Data Vault automation, considering
  WhereScape vs hand-coded ETL or dbt alternatives

  IMPLEMENTATION: Building WhereScape objects (Load/Stage/Hub/Link/Satellite),
  implementing custom templates, configuring scheduler jobs, writing metadata-driven
  transformations, integrating lineage with Collibra

  GUIDANCE: Asking about WhereScape best practices, template language patterns,
  scheduler configuration, Data Vault automation, medallion architecture mapping,
  lineage tracking capabilities, performance optimization

  REVIEW: Reviewing WhereScape object designs, checking template sprawl,
  validating scheduler dependencies, auditing version control integration,
  ensuring proper lineage metadata

  Do NOT use ali-wherescape for:
  - Generic ETL/ELT design (use ali-data-engineering instead)
  - Snowflake-specific SQL optimization (use ali-snowflake instead)
  - Collibra catalog administration (use ali-collibra instead)
  - Apache Airflow DAG development (use ali-apache-airflow instead)
---

# WhereScape Data Warehouse Automation

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing WhereScape-based data warehouse automation
- Planning template-based code generation strategies
- Architecting Data Vault automation with WhereScape
- Evaluating WhereScape vs alternatives (dbt, Informatica, hand-coded ETL)
- Planning lineage integration with Collibra or other data catalogs

**Implementation:**
- Building WhereScape objects (Load, Stage, Hub, Link, Satellite, Dimension, Fact)
- Implementing custom templates for code generation
- Configuring scheduler jobs and dependencies
- Writing metadata-driven transformations
- Integrating WhereScape lineage with external tools

**Guidance/Best Practices:**
- Asking about WhereScape template language
- Asking how to optimize WhereScape performance
- Asking about scheduler best practices
- Asking how to map WhereScape to medallion architecture
- Asking about version control integration patterns

**Review/Validation:**
- Reviewing WhereScape object designs
- Checking for template sprawl and complexity
- Validating scheduler job dependencies
- Auditing version control integration
- Ensuring lineage metadata is captured correctly

---

## Key Principles

- **Metadata-driven development**: WhereScape stores transformations as metadata, generates code from templates
- **Automation-first approach**: Drag-and-drop modeling generates SQL/stored procedures automatically
- **Built-in lineage**: Lineage captured automatically through metadata, available for export
- **Template customization**: Standard templates can be extended; avoid over-customization
- **Scheduler orchestration**: WhereScape scheduler manages dependencies; consider external orchestrators
- **Target platform agnostic**: Supports Snowflake, SQL Server, Oracle, Teradata, PostgreSQL
- **Data Vault acceleration**: Hub/Link/Satellite patterns automated through templates
- **Version control integration**: Export metadata to git; deployment via import/export

---

## WhereScape Product Suite

### WhereScape RED

**Purpose:** Data warehouse automation and transformation layer

**Capabilities:**
- Visual data modeling (drag-and-drop)
- Automatic code generation (SQL, stored procedures)
- Metadata repository
- Built-in scheduler
- Lineage tracking
- Template-based customization

**Target Platforms:** Snowflake, SQL Server, Oracle, Teradata, PostgreSQL, Amazon Redshift, Azure Synapse

**Primary Use Cases:**
- ETL/ELT pipeline automation
- Data warehouse layer development (STG, EDW)
- Data Vault 2.0 implementation
- Incremental loading patterns
- Slowly changing dimensions

### WhereScape 3D

**Purpose:** Data discovery, profiling, and source-to-target mapping

**Capabilities:**
- Source system profiling
- Data quality assessment
- Relationship discovery
- Automated mapping generation
- Documentation generation

**Integration:** Feeds metadata into WhereScape RED

**Primary Use Cases:**
- New source system onboarding
- Impact analysis before migrations
- Data quality baseline establishment

---

## WhereScape Object Types

| Object Type | Purpose | Common Use |
|-------------|---------|------------|
| **Load Table** | Extract from source to staging | Raw ingestion |
| **Stage Table** | Cleanse and validate | Bronze → Silver |
| **Hub** | Store business keys (Data Vault) | Unique entities |
| **Link** | Store relationships (Data Vault) | Entity connections |
| **Satellite** | Store descriptive attributes (Data Vault) | History tracking |
| **Dimension** | Business-ready attributes | Star schema |
| **Fact** | Business metrics | Star schema |
| **Aggregate** | Pre-calculated summaries | Performance |

**For detailed implementation examples with generated SQL, consult references/object-types.md covering:**
- Load Tables with incremental and CDC patterns
- Stage Tables with SCD Type 2 examples
- Hub/Link/Satellite with hash key generation
- Dimension tables with surrogate keys
- Fact tables with dimension lookups
- Aggregate tables with GROUP BY patterns

---

## Common Pitfalls

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Over-reliance on generated code | Hard to debug, complex SQL | Review generated SQL, simplify templates |
| Template sprawl | 50+ custom templates, unmaintainable | Use standard templates, customize only when needed |
| No version control | Lost changes, no audit trail | Export metadata to git regularly |
| Scheduler complexity | 1000+ jobs, dependency hell | Use external orchestrator (Airflow) |
| Ignoring generated SQL | Assume it works, find bugs in production | Always review generated SQL for new objects |
| Hardcoded values in templates | Not reusable | Use metadata variables |
| No testing before deployment | Import breaks production | Test imports in lower environments |
| Missing lineage metadata | Can't trace data flow | Configure source mappings correctly |
| Custom templates without docs | No one else understands them | Document template logic and purpose |
| No monitoring | Jobs fail silently | Integrate with monitoring (CloudWatch, Datadog) |

---

## Decision Guides

### Should We Use WhereScape?

**Consider WhereScape if:**
- Enterprise data warehouse with 100+ tables
- Data Vault 2.0 implementation
- Need automation for repetitive patterns
- Multi-platform support required (Oracle + Snowflake)
- Team has or can hire WhereScape expertise

**Avoid WhereScape if:**
- Small data warehouse (<50 tables)
- Cloud-native team (prefer dbt)
- Need native git integration
- Budget-constrained (high license cost)
- Complex custom transformations (hand-coded better)

### WhereScape RED vs WhereScape 3D

**Use WhereScape RED:**
- Build and automate data warehouse layers
- Generate ETL/ELT code
- Schedule and orchestrate jobs
- Implement Data Vault

**Use WhereScape 3D:**
- Discover source system structures
- Profile data quality
- Generate initial mappings
- Feed metadata into RED

**Combined approach:** Start with 3D for discovery, use RED for development and automation.

### Custom Templates vs Standard Templates

**Use custom templates when:**
- Company-specific standards (naming, audit columns)
- Complex hash key logic
- Special error handling requirements
- Integration with external systems

**Use standard templates when:**
- Standard SCD Type 1/2/3
- Simple incremental loading
- Basic fact/dimension patterns
- Data Vault Hub/Link/Satellite

**Rule of thumb:** Start with standard templates, customize only if needed. Track custom template count (goal: <10).

---

## Detailed References

For implementation details and examples, consult these reference files:

### references/object-types.md
Detailed SQL examples for all WhereScape object types:
- Load Tables (full, incremental, CDC)
- Stage Tables (SCD Type 1/2/3)
- Hub/Link/Satellite (Data Vault)
- Dimension tables (surrogate keys)
- Fact tables (dimension lookups)
- Aggregate tables (pre-calculated metrics)

### references/template-language.md
VBScript template development:
- Template structure and syntax
- Common template variables
- Metadata loops and parameterization
- Best practices for maintainability
- Template variable cheat sheet

### references/scheduler.md
Job scheduling and orchestration:
- Job types (Load, Update, Procedure, Script)
- Dependency configuration
- Error handling patterns
- External orchestrator integration (Airflow, Control-M, ADF)
- Job status meanings

### references/lineage.md
Lineage tracking and integration:
- Built-in lineage capabilities
- CSV/XML export formats
- Collibra integration approaches
- REST API push examples

### references/architecture-patterns.md
Architectural patterns and mappings:
- Medallion architecture (Bronze/Silver/Gold)
- Data Vault automation (Hub/Link/Satellite generation)
- Performance optimization (parallel loading, incremental patterns)
- Partition management

### references/version-control.md
Git integration and deployment:
- Metadata export/import
- Git workflow patterns
- Deployment procedures
- CLI commands

### references/alternatives-comparison.md
Detailed tool comparisons:
- WhereScape vs dbt
- WhereScape vs Informatica
- WhereScape vs hand-coded ETL

---

## Your Codebase

**Note:** Crown Equipment is evaluating WhereScape for data warehouse automation. No current implementation exists yet.

| Purpose | Location (Future) |
|---------|-------------------|
| WhereScape metadata exports | governance-rfp/wherescape/metadata/ |
| Custom templates | governance-rfp/wherescape/templates/ |
| Lineage export scripts | governance-rfp/wherescape/lineage/ |
| Deployment procedures | governance-rfp/wherescape/docs/deployment.md |

---

## References

- [WhereScape RED Documentation](https://www.wherescape.com/resources/documentation/)
- [WhereScape 3D Documentation](https://www.wherescape.com/resources/documentation/)
- [Data Vault 2.0 with WhereScape](https://datavaultalliance.com/)
- [Collibra Integration Guide](https://www.collibra.com/integrations/)
- [Crown Equipment Governance RFP](https://github.com/dmccarty-ali/Crown-Equipment-Requirements-Db) - REQ-F-100, REQ-F-101

---

**Document Version:** 2.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
