# ali-apache-iceberg - Catalog Configuration Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg catalog configuration details" or "I need Glue/Hive/REST/Nessie catalog examples"

---

## Hive Metastore Configuration (Spark)

```python
# spark-defaults.conf or Spark session config
spark.sql.catalog.my_catalog = org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.my_catalog.type = hive
spark.sql.catalog.my_catalog.uri = thrift://metastore-host:9083
spark.sql.catalog.my_catalog.warehouse = s3://my-bucket/warehouse

# Use catalog
spark.sql("USE my_catalog.bronze_layer")
spark.sql("SELECT * FROM cdc_customers")
```

**When to use:**
- Existing Hive ecosystem
- Wide compatibility across engines
- Battle-tested stability

**Limitations:**
- Single point of failure (requires HA setup)
- Not cloud-native
- Manual scaling

---

## AWS Glue Catalog Configuration

```python
# Spark configuration
spark.sql.catalog.glue_catalog = org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.glue_catalog.catalog-impl = org.apache.iceberg.aws.glue.GlueCatalog
spark.sql.catalog.glue_catalog.warehouse = s3://my-bucket/warehouse
spark.sql.catalog.glue_catalog.io-impl = org.apache.iceberg.aws.s3.S3FileIO

# Use Glue catalog
spark.sql("CREATE DATABASE IF NOT EXISTS glue_catalog.bronze_layer")
spark.sql("CREATE TABLE glue_catalog.bronze_layer.events (id BIGINT, data STRING) USING iceberg")
```

**When to use:**
- AWS-native deployments
- Serverless, no infrastructure management
- IAM integration for security

**Characteristics:**
- AWS-only (not multi-cloud)
- Eventual consistency model
- Automatic HA
- Pay-per-request pricing

---

## REST Catalog Configuration

```python
# REST catalog for multi-cloud or SaaS
spark.sql.catalog.rest_catalog = org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.rest_catalog.catalog-impl = org.apache.iceberg.rest.RESTCatalog
spark.sql.catalog.rest_catalog.uri = https://catalog.example.com/api/v1
spark.sql.catalog.rest_catalog.warehouse = s3://my-bucket/warehouse
spark.sql.catalog.rest_catalog.token = ${CATALOG_TOKEN}
```

**When to use:**
- Cloud-agnostic deployments
- Centralized catalog management
- Multi-region replication

**Characteristics:**
- Network dependency (requires internet/VPN)
- Vendor lock-in to catalog provider
- Simplified operations
- Built-in multi-tenancy

**Popular REST catalog providers:**
- Tabular (Iceberg-native SaaS)
- Polaris (open-source REST catalog)
- Nessie (git-like, supports REST API)

---

## Nessie Catalog Configuration (Git-like Versioning)

```python
# Nessie for multi-table transactions and branching
spark.sql.catalog.nessie = org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.nessie.catalog-impl = org.apache.iceberg.nessie.NessieCatalog
spark.sql.catalog.nessie.uri = http://nessie-server:19120/api/v1
spark.sql.catalog.nessie.ref = main  # branch name
spark.sql.catalog.nessie.warehouse = s3://my-bucket/warehouse

# Create branch for development
spark.sql("CREATE BRANCH dev_branch FROM main IN nessie")
spark.sql("USE BRANCH dev_branch IN nessie")
```

**When to use:**
- Git-like version control for data
- Multi-table transactions (atomic commits across tables)
- Development/staging/production branches
- Experimentation with rollback capability

**Unique features:**
- Branch and tag tables like git
- Merge branches (promote dev → prod)
- Time travel across entire catalog
- Audit history of all catalog changes

**Branch workflow example:**
```python
# Create feature branch
spark.sql("CREATE BRANCH feature_x FROM main IN nessie")

# Switch to feature branch
spark.sql("USE BRANCH feature_x IN nessie")

# Make changes to multiple tables
spark.sql("CREATE TABLE t1 ...")
spark.sql("INSERT INTO t2 ...")

# Merge back to main
spark.sql("MERGE BRANCH feature_x INTO main IN nessie")
```

---

## JDBC Catalog Configuration

```python
# JDBC catalog for small deployments
spark.sql.catalog.jdbc_catalog = org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.jdbc_catalog.catalog-impl = org.apache.iceberg.jdbc.JdbcCatalog
spark.sql.catalog.jdbc_catalog.uri = jdbc:postgresql://db-host:5432/iceberg_catalog
spark.sql.catalog.jdbc_catalog.jdbc.user = catalog_user
spark.sql.catalog.jdbc_catalog.jdbc.password = ${CATALOG_PASSWORD}
spark.sql.catalog.jdbc_catalog.warehouse = s3://my-bucket/warehouse
```

**When to use:**
- Small deployments (< 100 tables)
- Testing and development
- Any JDBC-compatible database available

**Limitations:**
- Not designed for HA (single database)
- Manual scaling required
- No native multi-region support

**Supported databases:**
- PostgreSQL
- MySQL
- SQLite (testing only)

---

## Catalog Comparison Summary

| Feature | Hive | Glue | REST | Nessie | JDBC |
|---------|------|------|------|--------|------|
| **HA** | Manual | Built-in | Provider-dependent | Yes (clustered) | No |
| **Multi-cloud** | Yes | No (AWS-only) | Yes | Yes | Yes |
| **Branching** | No | No | No | Yes | No |
| **Managed** | Self-hosted | Fully managed | Provider-managed | Self/Managed | Self-hosted |
| **Cost** | Infrastructure | Per-request | Subscription | Infrastructure | Infrastructure |
| **Best for** | Legacy Hive | AWS-native | SaaS simplicity | Git workflow | Testing |

---

## Migration Between Catalogs

### Glue → Hive Metastore

```python
# Export from Glue
tables = glue_client.get_tables(DatabaseName='bronze_layer')

# Import to Hive
for table in tables:
    hive_client.create_table(
        database='bronze_layer',
        table_name=table['Name'],
        location=table['StorageDescriptor']['Location']
    )
```

### Hive → Nessie

```python
# Register existing Iceberg tables in Nessie
nessie_catalog.register_table(
    table_identifier='bronze_layer.orders',
    metadata_location='s3://warehouse/bronze_layer/orders/metadata/v1.metadata.json'
)
```

**Note**: Catalog migration only updates metadata pointers. Data files remain unchanged.

---

## References

- [Iceberg Catalogs Documentation](https://iceberg.apache.org/docs/latest/configuration/)
- [AWS Glue Catalog Guide](https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/glue-catalog.html)
- [Nessie Documentation](https://projectnessie.org/docs/)
- [Polaris REST Catalog](https://www.polaris.io/)
