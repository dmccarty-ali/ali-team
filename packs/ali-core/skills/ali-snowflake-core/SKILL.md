---
name: ali-snowflake-core
description: |
  Snowflake fundamentals aligned with SnowPro Core certification (COF-C02). Use when:

  PLANNING: Designing warehouse sizing strategy, planning clustering keys, architecting
  multi-cluster warehouses, choosing between transient/permanent tables

  IMPLEMENTATION: Creating tables with VARIANT columns, writing COPY INTO statements,
  building Snowpipe configurations, implementing streams and tasks, writing UDFs

  GUIDANCE: Asking about micro-partition pruning, Time Travel retention, caching
  layers (metadata/result/warehouse), search optimization, materialized views

  REVIEW: Validating query performance with EXPLAIN, checking partition scan ratios,
  reviewing clustering depth, auditing security policies (row access, masking)

  Do NOT use for advanced Snowflake data engineering patterns like Snowpipe Streaming,
  lock-free staging, Snowpark, or Dynamic Tables (use ali-snowflake-data-engineer instead)
---

# Snowflake Core Fundamentals

## Overview

This skill provides foundational Snowflake knowledge organized around the six domains of the SnowPro Core Certification (COF-C02). It covers architecture, security, performance, data loading/unloading, transformations, and data protection/sharing.

**Certification Domain Weightings:**
| Domain | Weight |
|--------|--------|
| 1. Architecture & Features | 20-25% |
| 2. Account Access & Security | 20-25% |
| 3. Performance Concepts | 10-15% |
| 4. Data Loading & Unloading | 5-10% |
| 5. Data Transformations | 20-25% |
| 6. Data Protection & Sharing | 5-10% |

---

## Domain 1: Architecture & Features (20-25%)

### Three-Layer Architecture

Snowflake's unique architecture separates storage, compute, and services:

```
┌─────────────────────────────────────────────────────────┐
│                    CLOUD SERVICES                        │
│  Authentication, Metadata, Query Optimization, Security  │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│                   VIRTUAL WAREHOUSES                     │
│           (Compute Layer - Elastic, Independent)         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │   XS    │  │    S    │  │    M    │  │    L    │    │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘    │
└─────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────┐
│                   CENTRALIZED STORAGE                    │
│     (Cloud Storage - S3, Azure Blob, GCS)               │
│           Columnar, Compressed, Encrypted                │
└─────────────────────────────────────────────────────────┘
```

**Key Benefits:**
- **Elastic Compute:** Scale warehouses independently of storage
- **Workload Isolation:** Multiple warehouses can access same data simultaneously
- **No Contention:** Virtual warehouses don't compete for resources
- **Pay-per-use:** Compute billed only when running

### Micro-Partitions

Snowflake automatically organizes data into micro-partitions:

**Characteristics:**
- Size: 50-500 MB compressed (~10-20M rows uncompressed)
- Columnar storage format
- Immutable (copy-on-write for updates)
- Automatic compression
- Metadata stored for pruning optimization

**Metadata Tracked Per Micro-Partition:**
- Range of values for each column (MIN/MAX)
- Number of distinct values
- NULL count
- Compression information

**Pruning Example:**
```sql
-- Query with filter on date column
SELECT * FROM sales WHERE sale_date = '2024-01-15';

-- Snowflake checks micro-partition metadata
-- Only scans partitions where MIN(sale_date) <= '2024-01-15' <= MAX(sale_date)
-- Other partitions are "pruned" (skipped)
```

### Clustering

**Natural Clustering:** Data is clustered by insertion order
**Explicit Clustering Keys:** Define columns for optimal pruning

```sql
-- Add clustering key
ALTER TABLE sales CLUSTER BY (sale_date, region);

-- Check clustering status
SELECT SYSTEM$CLUSTERING_INFORMATION('sales');
```

**When to Use Clustering:**
- Tables > 1TB
- Frequently filtered columns not matching insert order
- Queries with range filters or equality predicates

**Clustering Depth:** Lower is better (measures how well-ordered data is)

### Editions

| Edition | Features |
|---------|----------|
| **Standard** | Core features, 1-day Time Travel |
| **Enterprise** | Multi-cluster warehouses, 90-day Time Travel, materialized views |
| **Business Critical** | Enhanced security, HIPAA/PCI compliance, failover |
| **Virtual Private** | Dedicated infrastructure, customer-managed keys |

### Tools & Interfaces

**Snowsight:** Web-based SQL editor and dashboard builder
**SnowSQL:** Command-line client
**Snowpark:** DataFrame API for Python, Java, Scala
**Connectors:** JDBC, ODBC, Python, Node.js, Go, .NET
**SQL API:** REST API for SQL execution

### Object Hierarchy

```
ORGANIZATION
└── ACCOUNT
    └── DATABASE
        └── SCHEMA
            ├── TABLE (permanent, temporary, transient)
            ├── VIEW (standard, secure, materialized)
            ├── STAGE (internal, external)
            ├── FILE FORMAT
            ├── PIPE
            ├── STREAM
            ├── TASK
            ├── FUNCTION (UDF, UDTF)
            ├── PROCEDURE
            └── SEQUENCE
```

---

## Domain 2: Account Access & Security (20-25%)

### Authentication Methods

**Password Authentication:** Basic username/password
**MFA:** Multi-factor authentication (Duo, RSA)
**Key Pair Authentication:** RSA key pairs for programmatic access
**SSO/SAML:** Federated authentication with identity providers
**OAuth:** External OAuth integration

**Key Pair Example:**
```sql
-- Assign public key to user
ALTER USER jsmith SET RSA_PUBLIC_KEY = 'MIIB...';

-- Connect with private key (SnowSQL)
snowsql -a account -u jsmith --private-key-path ~/.ssh/rsa_key.p8
```

### Role-Based Access Control (RBAC)

**System-Defined Roles:**
```
ORGADMIN          Organization-level administration
└── ACCOUNTADMIN  Full account control (use sparingly)
    └── SECURITYADMIN  Manage grants, roles, users
        └── SYSADMIN   Create databases, warehouses, schemas
            └── PUBLIC Default role for all users
```

**Custom Role Hierarchy:**
```sql
-- Create role hierarchy
CREATE ROLE analyst;
CREATE ROLE senior_analyst;
CREATE ROLE data_engineer;

-- Build hierarchy
GRANT ROLE analyst TO ROLE senior_analyst;
GRANT ROLE senior_analyst TO ROLE data_engineer;
GRANT ROLE data_engineer TO ROLE sysadmin;

-- Grant privileges
GRANT USAGE ON WAREHOUSE compute_wh TO ROLE analyst;
GRANT USAGE ON DATABASE analytics TO ROLE analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.public TO ROLE analyst;
```

### Privilege Inheritance

Privileges flow UP the role hierarchy:
- User in senior_analyst inherits all analyst privileges
- User in data_engineer inherits analyst + senior_analyst privileges

**Key Privileges:**
| Object | Privileges |
|--------|-----------|
| Warehouse | USAGE, OPERATE, MODIFY, MONITOR |
| Database | USAGE, CREATE SCHEMA, MONITOR |
| Schema | USAGE, CREATE TABLE/VIEW/PIPE/etc. |
| Table | SELECT, INSERT, UPDATE, DELETE, TRUNCATE |

### Network Policies

Restrict access by IP address:

```sql
-- Create network policy
CREATE NETWORK POLICY office_only
  ALLOWED_IP_LIST = ('192.168.1.0/24', '10.0.0.0/8')
  BLOCKED_IP_LIST = ('192.168.1.99');

-- Apply to account
ALTER ACCOUNT SET NETWORK_POLICY = office_only;

-- Apply to specific user
ALTER USER jsmith SET NETWORK_POLICY = office_only;
```

### Row-Level Security

Control data access at row level:

```sql
-- Create row access policy
CREATE ROW ACCESS POLICY region_filter AS (region VARCHAR) RETURNS BOOLEAN ->
  CASE
    WHEN CURRENT_ROLE() = 'ADMIN' THEN TRUE
    WHEN CURRENT_ROLE() = 'US_ANALYST' AND region = 'US' THEN TRUE
    ELSE FALSE
  END;

-- Apply to table
ALTER TABLE sales ADD ROW ACCESS POLICY region_filter ON (region);
```

### Column-Level Security (Masking)

Mask sensitive data based on role:

```sql
-- Create masking policy
CREATE MASKING POLICY ssn_mask AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('HR_ADMIN') THEN val
    ELSE '***-**-' || RIGHT(val, 4)
  END;

-- Apply to column
ALTER TABLE employees MODIFY COLUMN ssn SET MASKING POLICY ssn_mask;
```

### Object Tagging

Classify and manage objects with tags:

```sql
-- Create tag
CREATE TAG pii_classification ALLOWED_VALUES 'public', 'internal', 'confidential', 'restricted';

-- Apply to column
ALTER TABLE customers MODIFY COLUMN email SET TAG pii_classification = 'confidential';

-- Query tagged objects
SELECT * FROM TABLE(INFORMATION_SCHEMA.TAG_REFERENCES('pii_classification', 'COLUMN'));
```

---

## Domain 3: Performance Concepts (10-15%)

### Virtual Warehouse Sizing

| Size | Servers | Credits/Hour |
|------|---------|--------------|
| X-Small | 1 | 1 |
| Small | 2 | 2 |
| Medium | 4 | 4 |
| Large | 8 | 8 |
| X-Large | 16 | 16 |
| 2X-Large | 32 | 32 |
| 3X-Large | 64 | 64 |
| 4X-Large | 128 | 128 |

**Scaling Up:** More compute power per query (complex queries)
**Scaling Out:** More concurrent queries (multi-cluster warehouses)

### Multi-Cluster Warehouses (Enterprise+)

```sql
-- Create multi-cluster warehouse
CREATE WAREHOUSE analytics_wh
  WAREHOUSE_SIZE = MEDIUM
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 5
  SCALING_POLICY = 'STANDARD';  -- or 'ECONOMY'
```

**Scaling Policies:**
- **STANDARD:** Add clusters as soon as queue forms (minimize wait)
- **ECONOMY:** Add clusters only when queue persists 6+ minutes (save credits)

### Query Profile

Analyze query performance in Snowsight:

**Key Metrics:**
- **Bytes Scanned:** Data read from storage
- **Partitions Scanned vs Total:** Pruning effectiveness
- **Spillage to Local/Remote:** Memory exceeded, spilled to disk
- **Network:** Data transferred between nodes
- **Remote Disk I/O:** Worst case - data spilled to remote storage

**Common Performance Issues:**
```
Problem: High partitions scanned
Solution: Add clustering key on filter columns

Problem: Spillage to local disk
Solution: Use larger warehouse or optimize query

Problem: Spillage to remote disk
Solution: Significantly larger warehouse needed

Problem: Exploding JOINs
Solution: Review join conditions, add filters early
```

### Caching Layers

**Three Cache Levels:**

1. **Metadata Cache (Cloud Services):**
   - MIN/MAX values, row counts, distinct values
   - Used for: COUNT(*), MIN(), MAX() on full table
   - Free, always available

2. **Result Cache (Cloud Services):**
   - Stores query results for 24 hours
   - Same query, same role = instant results
   - No warehouse required
   - Invalidated on underlying data change

3. **Warehouse Cache (Local Disk):**
   - Raw data cached on warehouse SSD
   - Persists while warehouse running
   - Lost on suspend (after AUTO_SUSPEND period)

```sql
-- Check if query used result cache
SELECT query_id, bytes_scanned, percentage_scanned_from_cache
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE query_text ILIKE '%your query%';
```

### Query Acceleration Service

Offloads portions of eligible queries to shared compute:

```sql
-- Enable on warehouse
ALTER WAREHOUSE analytics_wh
  SET ENABLE_QUERY_ACCELERATION = TRUE
  QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8;
```

**Good candidates:** Queries with selective filters on large tables

### Search Optimization Service

Improves point lookup performance:

```sql
-- Enable for specific columns
ALTER TABLE customers ADD SEARCH OPTIMIZATION ON EQUALITY(customer_id);
ALTER TABLE logs ADD SEARCH OPTIMIZATION ON SUBSTRING(message);
```

**Use cases:**
- Equality predicates on high-cardinality columns
- IN lists with many values
- LIKE patterns with wildcards

### Materialized Views

Pre-computed query results automatically maintained:

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW daily_sales_mv AS
SELECT sale_date, region, SUM(amount) as total
FROM sales
GROUP BY sale_date, region;

-- Snowflake auto-maintains when base table changes
-- Queries auto-rewrite to use MV when beneficial
```

**Costs:** Storage + background maintenance compute

---

## Domain 4: Data Loading & Unloading (5-10%)

### Stages

**Internal Stages:**
```sql
-- User stage (automatic)
PUT file://local/path/file.csv @~

-- Table stage (automatic per table)
PUT file://local/path/file.csv @%my_table

-- Named internal stage
CREATE STAGE my_stage;
PUT file://local/path/file.csv @my_stage;
```

**External Stages:**
```sql
-- S3
CREATE STAGE s3_stage
  URL = 's3://bucket/path/'
  STORAGE_INTEGRATION = s3_int;

-- Azure
CREATE STAGE azure_stage
  URL = 'azure://account.blob.core.windows.net/container/path/'
  STORAGE_INTEGRATION = azure_int;

-- GCS
CREATE STAGE gcs_stage
  URL = 'gcs://bucket/path/'
  STORAGE_INTEGRATION = gcs_int;
```

### File Formats

```sql
CREATE FILE FORMAT my_csv_format
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  NULL_IF = ('NULL', 'null', '')
  EMPTY_FIELD_AS_NULL = TRUE
  ERROR_ON_COLUMN_COUNT_MISMATCH = FALSE;

CREATE FILE FORMAT my_json_format
  TYPE = 'JSON'
  STRIP_OUTER_ARRAY = TRUE;

CREATE FILE FORMAT my_parquet_format
  TYPE = 'PARQUET';
```

### COPY INTO (Loading)

```sql
-- Basic load
COPY INTO my_table
FROM @my_stage
FILE_FORMAT = my_csv_format;

-- With transformation
COPY INTO my_table (col1, col2, col3)
FROM (
  SELECT $1, $2, CURRENT_TIMESTAMP()
  FROM @my_stage
)
FILE_FORMAT = my_csv_format;

-- Load specific files
COPY INTO my_table
FROM @my_stage
FILES = ('file1.csv', 'file2.csv');

-- Pattern matching
COPY INTO my_table
FROM @my_stage
PATTERN = '.*2024.*\\.csv';
```

**Key Options:**
| Option | Purpose |
|--------|---------|
| ON_ERROR | CONTINUE, SKIP_FILE, ABORT_STATEMENT |
| FORCE | Reload files even if previously loaded |
| VALIDATION_MODE | RETURN_ERRORS, RETURN_n_ROWS (dry run) |
| PURGE | Delete files after successful load |

### Snowpipe (Continuous Loading)

```sql
-- Create pipe
CREATE PIPE my_pipe
  AUTO_INGEST = TRUE
AS
COPY INTO my_table
FROM @my_external_stage
FILE_FORMAT = my_csv_format;

-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('my_pipe');

-- Manual refresh
ALTER PIPE my_pipe REFRESH;
```

**Auto-ingest:** Cloud event notifications trigger loading

### COPY INTO (Unloading)

```sql
-- Unload to stage
COPY INTO @my_stage/export/
FROM my_table
FILE_FORMAT = (TYPE = 'CSV' HEADER = TRUE)
OVERWRITE = TRUE
SINGLE = FALSE
MAX_FILE_SIZE = 100000000;

-- Unload with query
COPY INTO @my_stage/export/
FROM (SELECT * FROM my_table WHERE region = 'US')
FILE_FORMAT = (TYPE = 'PARQUET');
```

### GET (Download from Internal Stage)

```sql
-- Download files
GET @my_stage/file.csv file://local/path/;
```

### External Tables

Query files without loading:

```sql
CREATE EXTERNAL TABLE ext_sales (
  sale_date DATE AS (VALUE:sale_date::DATE),
  amount NUMBER AS (VALUE:amount::NUMBER)
)
WITH LOCATION = @my_external_stage
FILE_FORMAT = my_parquet_format
AUTO_REFRESH = TRUE;
```

---

## Domain 5: Data Transformations (20-25%)

### Sampling

```sql
-- Fraction-based (percentage)
SELECT * FROM large_table SAMPLE (10);  -- ~10% of rows

-- Fixed-size (ROWS)
SELECT * FROM large_table SAMPLE (1000 ROWS);

-- Block-level sampling (faster for large tables)
SELECT * FROM large_table SAMPLE BLOCK (10);

-- Repeatable sampling
SELECT * FROM large_table SAMPLE (10) SEED (42);
```

### Semi-Structured Data

**VARIANT Data Type:** Stores JSON, Avro, Parquet, ORC, XML

```sql
-- Create table with VARIANT
CREATE TABLE events (
  event_id INT,
  event_data VARIANT
);

-- Insert JSON
INSERT INTO events VALUES (1, PARSE_JSON('{"user": "john", "action": "click"}'));

-- Query with dot notation
SELECT event_data:user::STRING, event_data:action::STRING
FROM events;

-- Query nested data
SELECT event_data:items[0]:name::STRING
FROM events;
```

### FLATTEN

Expand arrays and objects into rows:

```sql
-- Array flattening
SELECT
  e.event_id,
  f.value:item_id::INT as item_id,
  f.value:quantity::INT as quantity
FROM events e,
LATERAL FLATTEN(input => e.event_data:items) f;

-- Object flattening
SELECT
  f.key,
  f.value
FROM events e,
LATERAL FLATTEN(input => e.event_data) f;
```

**FLATTEN Output Columns:**
- SEQ: Sequence number
- KEY: Object key (null for arrays)
- PATH: Path to element
- INDEX: Array index
- VALUE: Element value
- THIS: Input value

### Common Semi-Structured Functions

```sql
-- Type checking
TYPEOF(variant_col)                    -- Returns data type
IS_INTEGER(variant_col)                -- Boolean check
IS_ARRAY(variant_col)

-- Array functions
ARRAY_SIZE(array_col)                  -- Count elements
ARRAY_CONTAINS(value, array_col)       -- Check membership
ARRAY_AGG(col)                         -- Aggregate to array
ARRAY_CONSTRUCT(a, b, c)               -- Build array

-- Object functions
OBJECT_KEYS(object_col)                -- Get key names
OBJECT_CONSTRUCT('key1', val1, ...)    -- Build object
GET(variant_col, 'path')               -- Get nested value
```

### Streams (Change Data Capture)

Track changes to tables:

```sql
-- Create stream
CREATE STREAM sales_stream ON TABLE sales;

-- Query stream (shows changes since last consume)
SELECT * FROM sales_stream;

-- Stream columns
-- METADATA$ACTION: INSERT, DELETE
-- METADATA$ISUPDATE: TRUE if UPDATE (shows as DELETE + INSERT)
-- METADATA$ROW_ID: Unique row identifier

-- Consume stream in DML
INSERT INTO sales_history
SELECT *, CURRENT_TIMESTAMP() as loaded_at
FROM sales_stream;
-- Stream is now empty until next changes
```

**Stream Types:**
- **Standard:** Tracks INSERT, UPDATE, DELETE
- **Append-only:** Tracks only INSERTs (more efficient)

### Tasks (Scheduling)

Automate SQL execution:

```sql
-- Create task
CREATE TASK daily_aggregation
  WAREHOUSE = compute_wh
  SCHEDULE = 'USING CRON 0 1 * * * America/New_York'  -- 1 AM daily
AS
INSERT INTO daily_summary
SELECT DATE(order_date), SUM(amount)
FROM orders
WHERE order_date = CURRENT_DATE() - 1
GROUP BY 1;

-- Task using stream
CREATE TASK process_changes
  WAREHOUSE = compute_wh
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('sales_stream')
AS
INSERT INTO sales_history
SELECT * FROM sales_stream;

-- Task dependencies (DAG)
CREATE TASK child_task
  WAREHOUSE = compute_wh
  AFTER parent_task
AS
CALL my_procedure();

-- Enable task
ALTER TASK daily_aggregation RESUME;
```

### User-Defined Functions (UDFs)

```sql
-- SQL UDF
CREATE FUNCTION add_tax(amount FLOAT, rate FLOAT)
RETURNS FLOAT
AS 'amount * (1 + rate)';

-- JavaScript UDF
CREATE FUNCTION parse_email(email STRING)
RETURNS VARIANT
LANGUAGE JAVASCRIPT
AS $$
  var parts = EMAIL.split('@');
  return {user: parts[0], domain: parts[1]};
$$;

-- Python UDF
CREATE FUNCTION sentiment(text STRING)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
PACKAGES = ('textblob')
HANDLER = 'analyze'
AS $$
from textblob import TextBlob
def analyze(text):
    return 'positive' if TextBlob(text).sentiment.polarity > 0 else 'negative'
$$;
```

### Stored Procedures

```sql
-- SQL procedure
CREATE PROCEDURE update_stats(table_name VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS
BEGIN
  EXECUTE IMMEDIATE 'ALTER TABLE ' || table_name || ' REFRESH';
  RETURN 'Stats updated for ' || table_name;
END;

-- JavaScript procedure
CREATE PROCEDURE process_batch()
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
EXECUTE AS CALLER
AS $$
  var stmt = snowflake.createStatement({sqlText: "SELECT COUNT(*) FROM my_table"});
  var rs = stmt.execute();
  rs.next();
  return "Count: " + rs.getColumnValue(1);
$$;

-- Python procedure
CREATE PROCEDURE load_and_transform()
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
PACKAGES = ('snowflake-snowpark-python')
HANDLER = 'main'
AS $$
def main(session):
    df = session.table("raw_data")
    df.write.save_as_table("processed_data", mode="overwrite")
    return "Success"
$$;
```

---

## Domain 6: Data Protection & Sharing (5-10%)

### Time Travel

Query and restore historical data:

```sql
-- Query as of timestamp
SELECT * FROM sales AT (TIMESTAMP => '2024-01-15 10:30:00');

-- Query as of offset
SELECT * FROM sales AT (OFFSET => -60*60);  -- 1 hour ago

-- Query as of statement
SELECT * FROM sales BEFORE (STATEMENT => '01a1b2c3-...');

-- Restore dropped table
UNDROP TABLE accidentally_dropped;

-- Clone from point in time
CREATE TABLE sales_backup CLONE sales AT (TIMESTAMP => '2024-01-15 10:00:00');
```

**Time Travel Retention:**
- Standard Edition: 0-1 days
- Enterprise+: 0-90 days
- Transient/Temporary tables: 0-1 days max

```sql
-- Set retention period
ALTER TABLE sales SET DATA_RETENTION_TIME_IN_DAYS = 30;
```

### Fail-Safe

**Purpose:** 7-day recovery by Snowflake Support only (after Time Travel expires)
**Cannot be disabled or queried**
**Enterprise+ feature**

```
Timeline:
[Active Data] → [Time Travel: 1-90 days] → [Fail-Safe: 7 days] → [Purged]
```

### Cloning (Zero-Copy)

Instant metadata-only copies:

```sql
-- Clone table
CREATE TABLE sales_dev CLONE sales;

-- Clone schema
CREATE SCHEMA dev_schema CLONE prod_schema;

-- Clone database
CREATE DATABASE dev_db CLONE prod_db;

-- Clone at point in time
CREATE TABLE sales_backup CLONE sales AT (TIMESTAMP => '2024-01-15 00:00:00');
```

**Characteristics:**
- Instant (metadata operation)
- No additional storage until divergence
- Independent after clone (changes don't affect original)

### Encryption

**Always-On Encryption:**
- Data encrypted at rest (AES-256)
- Data encrypted in transit (TLS 1.2+)
- Automatic key rotation (annual)

**Tri-Secret Secure (Business Critical+):**
- Customer-managed key + Snowflake key + composite key
- Customer can revoke access instantly

### Data Sharing

**Direct Sharing (Provider → Consumer):**
```sql
-- Provider account: Create share
CREATE SHARE sales_share;
GRANT USAGE ON DATABASE sales_db TO SHARE sales_share;
GRANT USAGE ON SCHEMA sales_db.public TO SHARE sales_share;
GRANT SELECT ON TABLE sales_db.public.transactions TO SHARE sales_share;

-- Add consumer account
ALTER SHARE sales_share ADD ACCOUNTS = consumer_account;

-- Consumer account: Create database from share
CREATE DATABASE shared_sales FROM SHARE provider_account.sales_share;
```

**Key Points:**
- Read-only for consumers
- No data copying (live access)
- Provider controls access
- Consumer pays for compute only

### Secure Views

Hide underlying data and logic:

```sql
CREATE SECURE VIEW customer_summary AS
SELECT customer_id, region, total_orders
FROM customers
WHERE region = CURRENT_USER_REGION();
```

**Secure views:**
- Query plan not visible to consumers
- Optimizer doesn't push predicates into view
- Required for sharing sensitive data

### Reader Accounts

Consumer accounts without Snowflake subscription:

```sql
-- Create reader account
CREATE MANAGED ACCOUNT reader_acct
  ADMIN_NAME = 'admin'
  ADMIN_PASSWORD = 'SecurePass123!'
  TYPE = READER;

-- Share data with reader account
ALTER SHARE my_share ADD ACCOUNTS = reader_acct;
```

**Provider pays for reader account compute**

### Snowflake Marketplace

- Public listings: Available to all Snowflake customers
- Private listings: Available to specific accounts
- Data products: Tables, views, functions, notebooks

---

## Quick Reference

### Common Commands

```sql
-- Account info
SELECT CURRENT_ACCOUNT(), CURRENT_REGION(), CURRENT_USER(), CURRENT_ROLE();

-- Warehouse management
ALTER WAREHOUSE wh RESUME;
ALTER WAREHOUSE wh SUSPEND;
ALTER WAREHOUSE wh SET WAREHOUSE_SIZE = MEDIUM;

-- Session settings
ALTER SESSION SET USE_CACHED_RESULT = FALSE;  -- Force fresh query
ALTER SESSION SET QUERY_TAG = 'debug_session';

-- Object information
SHOW TABLES IN SCHEMA my_schema;
DESCRIBE TABLE my_table;
SELECT GET_DDL('TABLE', 'my_table');
```

### Cost Management

```sql
-- Resource monitors
CREATE RESOURCE MONITOR monthly_limit
  WITH CREDIT_QUOTA = 1000
  TRIGGERS ON 75 PERCENT DO NOTIFY
           ON 100 PERCENT DO SUSPEND;

ALTER WAREHOUSE analytics_wh SET RESOURCE_MONITOR = monthly_limit;

-- Usage queries
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME > DATEADD('day', -7, CURRENT_TIMESTAMP());

SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY TOTAL_ELAPSED_TIME DESC
LIMIT 10;
```

---

## Related Skills

- **snowflake-data-engineer:** Advanced topics aligned with SnowPro Advanced: Data Engineer (DEA-C02)
- **snowflake-admin:** Administration topics aligned with SnowPro Advanced: Administrator

---

## Official Resources

- [Snowflake Documentation](https://docs.snowflake.com/)
- [SnowPro Core Certification](https://learn.snowflake.com/en/certifications/snowpro-core/)
- [Snowflake Community](https://community.snowflake.com/)
- [Snowflake Guides](https://guides.snowflake.com/)

---

**Document Version:** 1.0
**Last Updated:** 2026-01-01
**Based On:** SnowPro Core COF-C02 Exam Domains
**Maintained By:** Aliunde

**Sources:**
- [Snowflake Certifications](https://learn.snowflake.com/en/certifications/)
- [SnowPro Core COF-C02 Exam Syllabus](https://www.vmexam.com/snowflake/snowflake-snowpro-core-certification-exam-syllabus)
- [Snowflake Documentation](https://docs.snowflake.com/)
