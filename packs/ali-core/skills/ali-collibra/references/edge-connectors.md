# ali-collibra - Edge Connectors Reference

## Collibra Edge

### What is Edge?

Collibra Edge is an on-premises agent that scans metadata from sources behind firewalls and sends it to cloud Collibra.

**Architecture:**
```
[On-Prem Data Sources] ← JDBC/API ← [Collibra Edge Agent]
                                            ↓ HTTPS
                                     [Collibra Cloud SaaS]
```

### Supported Data Sources (JDBC)

| Database | Connection Type | Metadata Scanned |
|----------|----------------|------------------|
| **MySQL** | JDBC | Schemas, tables, columns, types, keys |
| **Oracle** | JDBC | Schemas, tables, columns, types, keys |
| **PostgreSQL** | JDBC | Schemas, tables, columns, types, keys |
| **SAP HANA** | JDBC | Schemas, tables, columns, types, keys |
| **Snowflake** | JDBC | Databases, schemas, tables, columns, types, keys |
| **SQL Server** | JDBC | Databases, schemas, tables, columns, types, keys |
| **IBM DB2** | JDBC | Schemas, tables, columns, types, keys |
| **Greenplum** | JDBC | Schemas, tables, columns, types, keys |
| **Apache Hive** | JDBC | Databases, tables, columns, partitions |
| **Netezza** | JDBC | Databases, schemas, tables, columns |
| **Spark SQL** | JDBC | Databases, tables, columns |

### Edge Capabilities

| Capability | Description |
|-----------|-------------|
| **Catalog Connectors** | Scan database schemas into Collibra Data Catalog |
| **Lineage Connectors** | Extract SQL lineage from queries, ETL tools |
| **Profiling** | Sample data to generate statistics (nulls, cardinality, patterns) |
| **Scheduled Scans** | Cron-based scheduling for incremental metadata updates |
| **Schema Drift Detection** | Alert on added/removed/modified columns, tables |
| **Secure Tunnel** | HTTPS connection to Collibra Cloud, no inbound ports |

### Configuration Pattern

```yaml
# edge-config.yaml (conceptual example)
connections:
  - name: snowflake_prod
    type: snowflake
    jdbc_url: jdbc:snowflake://account.snowflakecomputing.com
    username: ${SNOWFLAKE_USER}
    password: ${SNOWFLAKE_PASSWORD}
    warehouse: METADATA_WH
    database: PROD_DB
    schema: PUBLIC

scanners:
  - name: catalog_scan_snowflake
    connection: snowflake_prod
    type: catalog
    schedule: "0 2 * * *"  # Daily at 2 AM
    include_schemas:
      - LAKE_*
      - STG_*
      - EDW_*
    exclude_schemas:
      - TEMP_*

  - name: lineage_scan_snowflake
    connection: snowflake_prod
    type: lineage
    schedule: "0 3 * * *"  # Daily at 3 AM
    query_history_days: 7
```

### Schema Drift Alerts

Edge can detect and notify on schema changes:
- New tables added
- Tables dropped
- Columns added/removed
- Data types changed
- Primary/foreign keys modified

### Edge Configuration Files

```
# Edge installation directory structure
collibra-edge/
├── conf/
│   ├── application.yml       # Main configuration
│   ├── connections.yml        # Database connections
│   └── scanners.yml           # Scanner definitions
├── logs/
│   ├── edge.log               # Application logs
│   └── scanner.log            # Scanner execution logs
└── bin/
    └── edge.sh                # Start/stop script
```

### Deployment Models

| Model | Use Case | Characteristics |
|-------|----------|----------------|
| **Cloud SaaS** | Most common, fully managed | Collibra hosts, maintains, upgrades; multi-tenant |
| **On-Premises** | Regulatory or air-gapped environments | Customer hosts, manages, upgrades; single-tenant |
| **Hybrid** | Cloud Collibra + Edge agents | SaaS platform with on-prem metadata scanning |
