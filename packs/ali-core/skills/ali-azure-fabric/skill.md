---
name: ali-azure-fabric
description: |
  Microsoft Fabric platform fundamentals covering workspaces, Git integration,
  data engineering, analytics, and governance. Use when:

  PLANNING: Designing Fabric workspace architecture, planning Git integration
  with Azure DevOps, architecting data pipelines, considering deployment strategies,
  evaluating Fabric capacity sizing

  IMPLEMENTATION: Creating Fabric items (Dataflows, Pipelines, Warehouses, Lakehouses,
  Semantic Models, Reports), configuring Git integration, building ETL/ELT workflows,
  implementing medallion architecture, setting up CI/CD

  GUIDANCE: Asking about Fabric best practices, workspace organization, Git workflow
  patterns, capacity management, security and access control, deployment strategies,
  OneLake storage patterns

  REVIEW: Validating Fabric architecture, checking Git configuration, reviewing
  deployment processes, auditing workspace security, assessing disaster recovery
  readiness, evaluating cost optimization
---

# Microsoft Fabric

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Fabric workspace architecture and organization
- Planning Git integration with Azure DevOps or GitHub
- Architecting data engineering pipelines and workflows
- Considering deployment and promotion strategies across environments
- Evaluating Fabric capacity sizing and cost management
- Planning disaster recovery and backup strategies

**Implementation:**
- Creating Fabric workspace items (Dataflows Gen2, Data Pipelines, Warehouses, Lakehouses)
- Configuring workspace Git integration
- Implementing ETL/ELT workflows using Dataflow Gen2 or Data Pipelines
- Building medallion architecture (Bronze/Silver/Gold layers)
- Setting up environment promotion (Dev/Test/Prod)
- Implementing semantic models and Power BI reports

**Guidance/Best Practices:**
- Asking about Fabric workspace organization patterns
- Asking how to structure Git repositories for Fabric
- Asking about branching strategies and PR workflows
- Asking about OneLake storage optimization
- Asking about capacity management and cost control
- Asking about security, access control, and compliance

**Review/Validation:**
- Reviewing workspace architecture and organization
- Validating Git integration configuration
- Checking deployment and promotion processes
- Auditing workspace security and access policies
- Assessing disaster recovery and version control readiness
- Evaluating cost optimization opportunities

---

## Key Principles

**Workspace Organization:**
- Separate workspaces by environment (Dev, Test, Prod) for isolation
- One Git-connected workspace per environment (typically Dev)
- Workspace naming convention: ProjectName-Environment (e.g., SalesAnalytics-Dev)
- Organize related items within a workspace by functional area

**Git Integration Strategy:**
- Connect Dev workspace to Git for source control
- Use feature branches for development, main branch for deployments
- Manual promotion to Test/Prod via "Get latest" or export/import
- Document unsupported item types and manual backup strategy
- Establish branch policies (PR required, approvals, checks)

**Data Architecture:**
- Use Lakehouses for raw data storage (Bronze layer)
- Use Warehouses for structured analytics (Gold layer)
- OneLake as unified storage layer across all Fabric items
- Shortcuts for cross-workspace or external data access
- Medallion architecture: Bronze (raw) → Silver (cleansed) → Gold (curated)

**Deployment & Promotion:**
- Git for Dev workspace version control
- Manual promotion patterns: Git sync, export/import, or Deployment Pipelines (preview)
- Test manual deployments before production rollout
- Document deployment runbook with validation steps

**Security & Access:**
- Workspace roles: Admin, Member, Contributor, Viewer
- Item-level permissions for fine-grained access
- Service principal or managed identity for automation
- Conditional access policies may require configuration for Git integration
- Row-level security (RLS) in semantic models for data access control

---

## Fabric Architecture

### Core Components

```
Microsoft Fabric (SaaS Platform)
├── Workspaces (organizational containers)
│   ├── Dev Workspace (Git-connected)
│   ├── Test Workspace
│   └── Prod Workspace
│
├── OneLake (unified data lake, single storage layer)
│   ├── Lakehouse storage (Delta/Parquet format)
│   ├── Warehouse storage (SQL engine)
│   └── Shortcuts (external data references)
│
├── Data Engineering
│   ├── Dataflow Gen2 (low-code ETL)
│   ├── Data Pipeline (orchestration, ADF-like)
│   ├── Notebook (Spark, Python/Scala/R)
│   └── Spark Job Definition
│
├── Data Warehouse
│   ├── Warehouse (SQL analytics engine)
│   └── SQL Endpoint (read-only on Lakehouse)
│
├── Data Science
│   ├── ML Models
│   ├── Experiments
│   └── Notebooks
│
├── Real-Time Analytics
│   ├── Eventstream
│   ├── KQL Database
│   └── KQL Queryset
│
└── Power BI
    ├── Semantic Model (dataset)
    ├── Report
    ├── Dashboard
    └── Paginated Report
```

### Capacity & Licensing

**Capacity Units (CUs):**
- F2: 2 CUs (trial, dev)
- F4: 4 CUs
- F8: 8 CUs
- F16: 16 CUs
- F32+: Enterprise workloads

**Consumption:** Pay-per-use model based on operations performed

**Trial Capacity:** 60-day trial with limited capacity for testing

---

## Git Integration

### Supported Item Types (as of 2025)

**Fully Supported:**
- Dataflow Gen2
- Data Pipeline
- Warehouse (DDL only, not data)
- Notebook
- Spark Job Definition
- KQL Database (schema)
- KQL Queryset
- ML Model
- ML Experiment

**Partially Supported / Preview:**
- Semantic Model (PBIX or .bim format)
- Report (PBIX format, dependencies on semantic model)

**Not Supported:**
- Dashboard
- Paginated Report
- Eventstream
- Lakehouse (structure but not data files)
- Datamart

**Manual Backup Strategy for Unsupported Items:**
- Export PBIX files manually for dashboards/paginated reports
- Script out definitions where possible
- Document manual export/import procedures
- Store copies in separate repository or blob storage

### Git Workflow

**Setup:**
1. Create Azure DevOps project and Git repository
2. Configure branch policies (require PR, approvals)
3. In Fabric workspace settings, connect to Git repository
4. Select branch (typically `main` or `develop`)
5. Authorize Fabric to access Azure DevOps (user identity or app)

**Developer Workflow:**
```
1. Developer creates feature branch in Azure DevOps
2. In Fabric Dev workspace, switch to feature branch
3. Author changes (dataflows, pipelines, notebooks)
4. Commit changes to feature branch with message
5. Create PR in Azure DevOps, request review
6. After approval, merge PR to main branch
7. In Fabric workspace, sync from main branch to get updates
```

**Promotion to Test/Prod:**
- Manual approach: "Get latest" from Git in Test/Prod workspaces (if Git-connected)
- Export/import approach: Export items from Dev, import to Test/Prod
- Deployment Pipelines (Power BI-centric, limited to semantic models/reports)

### Branch Strategies

**GitFlow Pattern (Recommended):**
- `main` branch: production-ready code
- `develop` branch: integration branch
- `feature/*` branches: individual feature development
- `hotfix/*` branches: urgent production fixes

**Trunk-Based Pattern (Simpler):**
- `main` branch: all development
- Short-lived feature branches: merge quickly (< 2 days)
- Release tags for versioning

---

## Data Engineering Patterns

### Dataflow Gen2

Low-code ETL tool (Power Query interface):

**Use When:**
- Data transformations can be expressed via Power Query M
- Business users need to author transformations
- Source systems have connectors (SQL, REST API, files)
- Transformations are relatively simple (filter, join, aggregate)

**Output Options:**
- Lakehouse table (Delta format)
- Warehouse table (SQL engine)
- Azure SQL Database, Dataverse, etc.

**Best Practices:**
- Parameterize connection strings and file paths
- Enable staging for incremental refresh
- Use query folding where possible (push operations to source)
- Monitor refresh history for failures

### Data Pipeline

Orchestration tool similar to Azure Data Factory:

**Use When:**
- Complex orchestration logic required (loops, conditionals)
- Multiple activities need coordination (copy, notebook, stored proc)
- Need to invoke external systems (REST API, Azure Function)
- Scheduling with dependencies

**Common Activities:**
- Copy Data: Move data between sources and destinations
- Notebook: Execute Spark notebook
- Stored Procedure: Run SQL in Warehouse
- Dataflow: Trigger Dataflow Gen2 refresh
- Web: Call REST API
- ForEach: Loop over items
- If Condition: Branching logic

**Best Practices:**
- Use parameters for environment-specific values
- Implement error handling (retries, failure paths)
- Log activity outputs to monitoring table
- Version pipeline definitions in Git

### Lakehouse vs Warehouse

| Feature | Lakehouse | Warehouse |
|---------|-----------|-----------|
| Storage | OneLake (Delta/Parquet) | OneLake (columnar) |
| Query Engine | SQL Endpoint (read-only) | Full T-SQL engine |
| Data Ingestion | Files, notebooks, dataflows | COPY INTO, dataflows, pipelines |
| Use Case | Data lake, raw/semi-structured | Structured analytics, BI |
| Git Support | Structure only | DDL scripts |
| Performance | Good for big data scans | Optimized for analytics queries |

**Lakehouse Best Practices:**
- Organize by medallion layers: Bronze (raw), Silver (cleansed), Gold (curated)
- Use Delta format for ACID transactions and time travel
- Partition large tables by date or other high-cardinality columns
- Use shortcuts to reference external data without copying

**Warehouse Best Practices:**
- Use for structured, relational analytics
- Create views and stored procedures for reusable logic
- Optimize queries with statistics and indexing (limited in Fabric)
- Leverage Fabric's automatic query optimization

---

## Workspace Management

### Workspace Roles

| Role | Permissions |
|------|-------------|
| **Admin** | Full control, manage workspace, configure Git, assign roles |
| **Member** | Create/edit items, cannot manage workspace settings |
| **Contributor** | Create/edit items, cannot delete or manage workspace |
| **Viewer** | Read-only access to workspace items |

### Workspace Settings

**Git Integration:**
- Connect to Azure DevOps or GitHub
- Select organization, project, repository, branch
- Configure directory path within repository
- Manage commit and sync operations

**Licenses:**
- Assign workspace to Fabric capacity (F SKU)
- Trial capacity available for testing

**Data Sources:**
- Configure gateway connections for on-premises data
- Manage data source credentials

### Multi-Environment Strategy

**Dev Workspace:**
- Git-connected to feature branches
- Developers create and modify items
- Frequent commits and PRs

**Test Workspace:**
- Receives promoted changes from Dev
- QA team validates functionality
- May be Git-connected to main branch (manual sync)

**Prod Workspace:**
- Receives validated changes from Test
- Strict change control (manual deployment)
- Not Git-connected (or read-only sync from main)
- Full backups and disaster recovery

---

## Deployment Patterns

### Manual Promotion (Common)

**Process:**
1. Develop in Dev workspace, commit to Git
2. Export item definitions (JSON/config) from Dev workspace
3. Import item definitions to Test workspace
4. Validate functionality in Test
5. Export from Test, import to Prod
6. Validate in Prod

**Advantages:**
- Simple, no additional tools required
- Works for all item types (including unsupported in Git)
- Full control over what gets promoted

**Disadvantages:**
- Manual process, error-prone
- No automated validation
- Time-consuming for frequent deployments

### Git-Based Promotion (Recommended for Supported Items)

**Process:**
1. Develop in Dev workspace (Git-connected to feature branch)
2. Commit changes and create PR
3. Merge PR to main branch after review
4. In Test workspace (Git-connected to main), click "Get latest"
5. Validate in Test
6. In Prod workspace (Git-connected to main), click "Get latest"
7. Validate in Prod

**Advantages:**
- Version control for all changes
- PR review workflow enforces quality
- Audit trail via Git history
- Rollback capability

**Disadvantages:**
- Only works for Git-supported item types
- Manual sync step required (not automatic)
- Conditional access policies may complicate automation

### Deployment Pipelines (Power BI Premium/Fabric)

**Process:**
1. Configure Deployment Pipeline with Dev → Test → Prod stages
2. Assign workspaces to each stage
3. Deploy semantic models and reports through pipeline UI
4. Configure deployment rules (parameter overrides, data source mappings)

**Advantages:**
- Built-in Fabric feature (no external tools)
- Parameter overrides per environment
- Visual deployment history

**Disadvantages:**
- Limited to Power BI items (semantic models, reports)
- Does not support Dataflows, Pipelines, Warehouses, Notebooks
- Still in preview for some item types

### CI/CD with Azure DevOps (Advanced)

**Process:**
1. Git-connected Dev workspace commits changes
2. Azure DevOps pipeline triggers on PR merge
3. Pipeline uses Fabric REST API to:
   - Export item definitions from Dev
   - Validate item definitions (linting, schema checks)
   - Deploy to Test workspace
   - Run automated tests
   - Deploy to Prod on approval
4. Post-deployment validation

**Advantages:**
- Fully automated promotion
- Integrated testing and validation
- Supports complex deployment logic
- Audit and compliance via pipeline logs

**Disadvantages:**
- Requires Fabric REST API expertise
- Complex setup and maintenance
- Service principal authentication required
- Limited documentation for Fabric API (evolving)

---

## Security & Access Control

### Authentication Options

**User Identity:**
- Default for Git integration
- Each developer uses their own credentials
- Suitable for small teams

**Service Principal:**
- App registration in Azure AD
- Recommended for automation and CI/CD
- Requires Fabric Admin to grant permissions

**Managed Identity:**
- System-assigned or user-assigned identity
- Used by Azure resources (VMs, Functions, Logic Apps)
- No credential management required

### Conditional Access Considerations

**Potential Blockers:**
- MFA required for Git access
- Trusted device policies
- IP restrictions

**Mitigations:**
- Whitelist Fabric service principal IP ranges
- Use Personal Access Tokens (PAT) if user identity blocked
- Work with security team to configure exceptions

### Row-Level Security (RLS)

Defined in semantic models to restrict data access:

```dax
[Territory] = USERNAME()
```

**Best Practices:**
- Define security roles in semantic model
- Map Azure AD users/groups to roles
- Test with "View As" feature in Power BI
- Document RLS rules in workspace documentation

---

## Disaster Recovery & Backup

### Git as Backup (Supported Items)

**What's Protected:**
- Dataflow Gen2 definitions (JSON)
- Data Pipeline definitions (JSON)
- Notebook code (.ipynb)
- Warehouse DDL scripts (.sql)
- Git history provides point-in-time recovery

**Recovery Process:**
1. Create new workspace or delete items
2. Connect to Git repository
3. Sync from desired branch/commit
4. Items are restored

### Manual Backup (Unsupported Items)

**PBIX Files (Reports, Semantic Models):**
- Download .pbix from Fabric workspace
- Store in separate Git repository or blob storage
- Version control via file naming (e.g., `report_v1.2.pbix`)

**Dashboards & Paginated Reports:**
- No export capability currently
- Screenshot documentation
- Recreate from semantic models if lost

### OneLake Data Backup

**Data Files:**
- OneLake stores data in Delta/Parquet format
- Enable OneLake data redundancy (LRS, GRS)
- Use Azure Blob lifecycle policies for archival
- Shortcuts allow referencing external backups

**Backup Strategy:**
- Schedule regular snapshots of critical Lakehouses
- Export Gold layer tables to Azure Blob Storage
- Document restore procedures in runbook

---

## Cost Optimization

### Capacity Management

**Right-Size Capacity:**
- Start with F2 or F4 for development
- Monitor capacity metrics (CPU, memory, throttling)
- Scale up to F8+ for production workloads
- Use Capacity Metrics app to identify bottlenecks

**Auto-Pause Capacity:**
- Pause dev/test capacities during off-hours
- Azure Automation runbook to schedule pause/resume
- Significant cost savings (capacity billed hourly)

### OneLake Storage Optimization

**Reduce Storage Costs:**
- Partition large tables to minimize scans
- Use Delta optimization (OPTIMIZE, VACUUM commands)
- Archive or delete obsolete data
- Use shortcuts instead of copying data

**Transaction Costs:**
- Minimize write operations (batch inserts vs row-by-row)
- Use Dataflow Gen2 staging to reduce warehouse writes
- Cache frequently accessed data in Gold layer

### Query Optimization

**Reduce Compute Usage:**
- Optimize SQL queries (avoid SELECT *, use indexes)
- Push filters to source systems (query folding)
- Use aggregations and cached results where possible
- Schedule batch jobs during low-cost periods

---

## Common Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| No Git integration | No version control or disaster recovery | Connect Dev workspace to Git, commit regularly |
| Single workspace for all environments | No isolation, promotes directly to production | Separate workspaces for Dev, Test, Prod |
| Hardcoded connection strings | Cannot promote across environments | Use parameters or environment variables |
| Git-connecting Prod workspace | Accidental overwrites from dev branches | Prod workspace not Git-connected, manual promotion |
| Ignoring unsupported item types | No backup for reports/dashboards | Manual export and storage in separate location |
| No branch policies | Unreviewed changes pushed to main | Require PRs and approvals in Azure DevOps |
| No capacity monitoring | Throttling and performance issues | Monitor Capacity Metrics app, set alerts |
| Large monolithic notebooks | Hard to maintain, slow execution | Break into smaller notebooks or parameterized functions |
| Copying data unnecessarily | Increased storage and compute costs | Use shortcuts to reference external data |
| No disaster recovery testing | Untested recovery procedures fail in crisis | Regularly test Git sync, manual restore procedures |

---

## Quick Reference

### Git Integration Commands

**Connect Workspace to Git:**
1. Workspace Settings → Git Integration
2. Select Azure DevOps or GitHub
3. Choose organization, project, repository, branch
4. Authorize access

**Commit Changes:**
1. Workspace → Source Control
2. Select uncommitted changes
3. Enter commit message
4. Commit

**Sync from Git:**
1. Workspace → Source Control
2. Click "Get latest" to pull changes from Git
3. Review incoming changes
4. Apply updates

### Dataflow Gen2 Quick Start

```powerquery
// Example: Load CSV from OneLake
let
    Source = Lakehouse.Contents(null),
    Files = Source{[workspaceId="workspace-guid"]}[Data],
    FilteredFiles = Table.SelectRows(Files, each [Name] = "sales.csv"),
    LoadedCSV = Csv.Document(FilteredFiles{0}[Content], [Delimiter=",", Encoding=65001]),
    PromotedHeaders = Table.PromoteHeaders(LoadedCSV)
in
    PromotedHeaders
```

### Data Pipeline Quick Start

**Copy Activity (File to Lakehouse):**
```json
{
  "type": "Copy",
  "source": {
    "type": "DelimitedTextSource",
    "filePath": "input/sales.csv"
  },
  "sink": {
    "type": "LakehouseTableSink",
    "tableName": "bronze_sales"
  }
}
```

### Warehouse DDL Examples

**Create Table:**
```sql
CREATE TABLE dbo.DimCustomer (
    CustomerKey INT IDENTITY(1,1) PRIMARY KEY,
    CustomerID NVARCHAR(50) NOT NULL,
    CustomerName NVARCHAR(255),
    Region NVARCHAR(50),
    LoadDate DATETIME2 DEFAULT GETDATE()
);
```

**Create View:**
```sql
CREATE VIEW dbo.vw_SalesSummary AS
SELECT
    c.Region,
    SUM(s.SalesAmount) AS TotalSales,
    COUNT(*) AS TransactionCount
FROM dbo.FactSales s
INNER JOIN dbo.DimCustomer c ON s.CustomerKey = c.CustomerKey
GROUP BY c.Region;
```

---

## References

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [Fabric Git Integration](https://learn.microsoft.com/en-us/fabric/cicd/git-integration/intro-to-git-integration)
- [OneLake Documentation](https://learn.microsoft.com/en-us/fabric/onelake/)
- [Fabric Capacity Management](https://learn.microsoft.com/en-us/fabric/enterprise/licenses)
- [Fabric REST API](https://learn.microsoft.com/en-us/rest/api/fabric/)
- [Azure DevOps Git Documentation](https://learn.microsoft.com/en-us/azure/devops/repos/git/)
