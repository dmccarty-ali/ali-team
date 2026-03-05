---
name: ali-snowflake-admin
description: |
  Snowflake operations with safety, cost, and security enforcement. Use when:

  PLANNING: Planning Snowflake infrastructure, warehouse sizing, role hierarchies,
  evaluating data sharing, considering environment separation, designing security architecture

  IMPLEMENTATION: Executing SnowSQL commands, creating/modifying objects, managing
  warehouses, configuring roles, loading data, granting privileges

  GUIDANCE: Asking about Snowflake best practices, cost optimization, credit
  consumption, role-based access control, zero-copy cloning, time travel

  REVIEW: Reviewing Snowflake configurations, auditing role grants, checking
  warehouse utilization, validating security settings
---

# Snowflake Admin

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Snowflake database architecture
- Planning warehouse sizing and auto-suspend strategies
- Designing role hierarchies and access control
- Evaluating data sharing options
- Planning environment separation (dev/staging/prod)
- Estimating costs and credit consumption

**Implementation:**
- Executing SnowSQL commands
- Creating or modifying databases, schemas, tables, views
- Managing warehouses (create, resize, suspend)
- Configuring roles and grants
- Loading data from stages
- Setting up shares and data exchange

**Guidance/Best Practices:**
- Cost optimization strategies
- Warehouse sizing recommendations
- Role-based access control patterns
- Zero-copy cloning vs backup strategies
- Time travel and fail-safe
- Query optimization for credit efficiency

**Review/Validation:**
- Auditing role grants and privilege escalation
- Reviewing warehouse utilization and cost
- Checking security configurations
- Validating compliance with governance policies

---

## Core Principles

### 1. Security First (Non-Negotiable)

| Rule | Rationale |
|------|-----------|
| **Never use ACCOUNTADMIN for routine work** | God-mode role - use only for user/role management |
| **Least privilege grants** | Start with minimal privileges, add only what's needed |
| **Role hierarchy enforcement** | ACCOUNTADMIN > SECURITYADMIN > SYSADMIN > custom roles |
| **Network policies required** | IP whitelisting for production accounts |
| **No GRANT ALL** | Always grant specific privileges |
| **Monitor failed login attempts** | Detect unauthorized access attempts |

### 2. Cost Consciousness

| Practice | Impact |
|----------|--------|
| **Right-size warehouses** | Start with XSMALL/SMALL, scale up when needed |
| **Enable auto-suspend** | 1-5 minutes for dev, 5-10 minutes for prod |
| **Use multi-cluster only when needed** | Auto-scaling costs add up quickly |
| **Cluster keys for large tables** | Improves query performance, reduces credits |
| **Materialized views sparingly** | Maintenance costs can exceed benefits |
| **Monitor query patterns** | Identify expensive queries early |

### 3. Environment Awareness

| Environment | Constraints |
|-------------|-------------|
| **dev** | XSMALL/SMALL warehouses, time travel 1-7 days, can be dropped |
| **staging** | SMALL/MEDIUM warehouses, production-like config, limited retention |
| **prod** | Sized appropriately, time travel 90 days, change management required |

**Environment Indicators:**
- Database names: `MYAPP_DEV`, `MYAPP_STAGING`, `MYAPP_PROD`
- Warehouse names: `DEV_WH`, `STAGING_WH`, `PROD_WH`
- Role names: `DEV_ADMIN`, `PROD_READ_ONLY`

---

## Dangerous Operations (REQUIRE CONFIRMATION)

### Critical Risk (Data Loss)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| DROP DATABASE | Permanent data loss | "Yes, drop database [name]" |
| DROP SCHEMA | Permanent data loss | "Yes, drop schema [name]" |
| DROP TABLE | Data loss (time travel recoverable) | "Yes, drop table [name]" |
| DROP VIEW | Logic loss | "Yes, drop view [name]" |
| TRUNCATE TABLE | Data loss (time travel recoverable) | "Yes, truncate table [name]" |
| DROP WAREHOUSE | Service disruption | "Yes, drop warehouse [name]" |
| CREATE OR REPLACE | Overwrites existing object | "Yes, replace [type] [name]" |

### Critical Risk (Security)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| GRANT ACCOUNTADMIN | God-mode access | "Yes, grant ACCOUNTADMIN to [user]" |
| GRANT SECURITYADMIN | Security admin access | "Yes, grant SECURITYADMIN to [user]" |
| GRANT SYSADMIN | System admin access | "Yes, grant SYSADMIN to [user]" |
| ALTER USER password | Credential change | "Yes, alter user [name]" |
| DROP USER | Account deletion | "Yes, drop user [name]" |
| DROP ROLE | Role deletion | "Yes, drop role [name]" |
| REVOKE on production | Access removal | "Yes, revoke [privilege] on [object]" |

### High Risk (Cost)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| CREATE WAREHOUSE XLARGE+ | High cost | "Yes, create [size] warehouse (~$X/hour)" |
| ALTER WAREHOUSE size to XLARGE+ | Cost increase | "Yes, resize to [size] (~$X/hour)" |
| Multi-cluster warehouse | Auto-scaling cost | "Yes, enable multi-cluster" |

---

## Safe Operations (No Confirmation Needed)

```sql
-- Metadata inspection
SHOW DATABASES;
SHOW SCHEMAS IN DATABASE db_name;
SHOW TABLES IN SCHEMA schema_name;
SHOW VIEWS IN SCHEMA schema_name;
SHOW WAREHOUSES;
SHOW USERS;
SHOW ROLES;
SHOW GRANTS ON DATABASE db_name;
SHOW GRANTS TO USER username;
SHOW GRANTS TO ROLE rolename;

-- Object description
DESCRIBE DATABASE db_name;
DESCRIBE SCHEMA schema_name;
DESCRIBE TABLE table_name;
DESCRIBE VIEW view_name;
DESCRIBE WAREHOUSE warehouse_name;

-- Data queries (read-only)
SELECT * FROM table_name LIMIT 100;
SELECT COUNT(*) FROM table_name;
EXPLAIN SELECT ... ;

-- Context switching
USE DATABASE db_name;
USE SCHEMA schema_name;
USE WAREHOUSE warehouse_name;
USE ROLE role_name;

-- Stage operations (read)
LIST @stage_name;
LIST @stage_name/path/;
```

---

## Warehouse Sizing Cost Reference

Always consider cost when creating or resizing warehouses:

| Size | Credits/Hour | Est. Monthly Cost | Use Case |
|------|--------------|-------------------|----------|
| XSMALL | 1 | ~$288 | Dev, small queries, < 100 concurrent users |
| SMALL | 2 | ~$576 | Testing, medium queries, 100-200 users |
| MEDIUM | 4 | ~$1,152 | Production (small workloads), 200-400 users |
| LARGE | 8 | ~$2,304 | Production (medium workloads), 400-800 users |
| XLARGE | 16 | ~$4,608 | Production (large workloads), heavy ETL |
| 2XLARGE | 32 | ~$9,216 | Very large data processing |
| 3XLARGE | 64 | ~$18,432 | Extreme workloads, complex analytics |
| 4XLARGE | 128 | ~$36,864 | Massive data processing |

**Notes:**
- Costs assume $0.40/credit (varies by region and contract)
- Monthly cost assumes 24/7 operation with no auto-suspend
- **Always enable auto-suspend** to reduce costs
- Multi-cluster warehouses multiply these costs by cluster count

**Cost Optimization Tips:**
- Start small, scale up based on actual performance metrics
- Use query profiling to identify bottlenecks before scaling
- Enable auto-suspend (1-5 min for dev, 5-10 min for prod)
- Consider separate warehouses for different workloads
- Monitor credit consumption with ACCOUNT_USAGE views

---

## Role Hierarchy Best Practices

### Standard Snowflake Role Hierarchy

```
ACCOUNTADMIN (top-level, use sparingly)
    ↓
SECURITYADMIN (manage users, roles, network policies)
    ↓
SYSADMIN (create databases, warehouses, schemas)
    ↓
Custom Functional Roles (app-specific privileges)
    ↓
PUBLIC (default role, minimal privileges)
```

### Recommended Custom Role Structure

```sql
-- Admin roles
CREATE ROLE DATA_ENGINEER_ADMIN;  -- Can create objects
CREATE ROLE DATA_ANALYST_ADMIN;   -- Can modify objects

-- Read-write roles
CREATE ROLE APP_WRITE;            -- Insert/update/delete data
CREATE ROLE APP_READ_WRITE;       -- Select/insert/update

-- Read-only roles
CREATE ROLE APP_READ_ONLY;        -- Select only
CREATE ROLE ANALYTICS_READ;       -- BI tool access

-- Grant hierarchy
GRANT ROLE DATA_ENGINEER_ADMIN TO ROLE SYSADMIN;
GRANT ROLE APP_WRITE TO ROLE DATA_ENGINEER_ADMIN;
GRANT ROLE APP_READ_ONLY TO ROLE APP_WRITE;
```

### Role Assignment Pattern

```sql
-- Grant database privileges to role
GRANT USAGE ON DATABASE myapp_prod TO ROLE app_read_only;
GRANT USAGE ON SCHEMA myapp_prod.public TO ROLE app_read_only;
GRANT SELECT ON ALL TABLES IN SCHEMA myapp_prod.public TO ROLE app_read_only;
GRANT SELECT ON FUTURE TABLES IN SCHEMA myapp_prod.public TO ROLE app_read_only;

-- Grant warehouse access
GRANT USAGE ON WAREHOUSE prod_wh TO ROLE app_read_only;

-- Grant role to user
GRANT ROLE app_read_only TO USER john_doe;
```

---

## Security Best Practices

### Network Policies

```sql
-- Create network policy (IP whitelist)
CREATE NETWORK POLICY office_and_vpn
  ALLOWED_IP_LIST = ('192.168.1.0/24', '10.0.0.0/8')
  BLOCKED_IP_LIST = ();

-- Apply to account
ALTER ACCOUNT SET NETWORK_POLICY = office_and_vpn;

-- Apply to specific user
ALTER USER sensitive_user SET NETWORK_POLICY = office_and_vpn;
```

### Password Policies

```sql
-- Set password policy
ALTER ACCOUNT SET
  MIN_PASSWORD_LENGTH = 12
  MAX_PASSWORD_AGE_DAYS = 90
  PASSWORD_COMPLEXITY = 'STRONG'
  MUST_CHANGE_PASSWORD = TRUE;
```

### Multi-Factor Authentication

```sql
-- Require MFA for user
ALTER USER admin_user SET MINS_TO_BYPASS_MFA = 0;

-- Disable MFA bypass window
ALTER ACCOUNT SET MINS_TO_BYPASS_MFA = 0;
```

---

## Time Travel and Fail-Safe

### Time Travel

Data recovery for dropped/modified objects:

| Edition | Time Travel Retention |
|---------|----------------------|
| Standard | 1 day (default, up to 1 day configurable) |
| Enterprise | 90 days (configurable) |

**Usage:**

```sql
-- Recover dropped table
UNDROP TABLE my_table;

-- Query historical data
SELECT * FROM my_table AT (TIMESTAMP => '2024-01-15 10:00:00'::timestamp);

-- Query before specific statement
SELECT * FROM my_table BEFORE (STATEMENT => '01a1b2c3-d4e5-f6g7-h8i9-j0k1l2m3n4o5');

-- Clone table from specific point
CREATE TABLE recovered_table CLONE my_table AT (TIMESTAMP => '2024-01-15 10:00:00'::timestamp);
```

### Fail-Safe

Additional 7-day recovery period after time travel expires (Enterprise only).

**Important:** Fail-safe data can only be recovered by Snowflake Support (not self-service).

---

## Zero-Copy Cloning

Create instant copies of databases, schemas, or tables without duplicating storage:

```sql
-- Clone entire database
CREATE DATABASE myapp_dev CLONE myapp_prod;

-- Clone schema
CREATE SCHEMA analytics_snapshot CLONE analytics;

-- Clone table
CREATE TABLE staging_data CLONE production_data;

-- Clone from specific point in time
CREATE TABLE backup_table CLONE source_table AT (TIMESTAMP => '2024-01-15 10:00:00'::timestamp);
```

**Use Cases:**
- Dev/test environments from production
- Quick backups before risky changes
- Experimentation without affecting production
- QA environments with production-like data

**Cost:** Only additional storage for changes to cloned objects (not original data).

---

## Data Loading Patterns

### Stage-Based Loading

```sql
-- Create internal stage
CREATE STAGE my_stage;

-- Upload files via SnowSQL
PUT file:///path/to/local/file.csv @my_stage;

-- List staged files
LIST @my_stage;

-- Load from stage
COPY INTO my_table
  FROM @my_stage/file.csv
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"')
  ON_ERROR = 'CONTINUE';

-- Remove staged files after load
REMOVE @my_stage/file.csv;
```

### External Stage (S3)

```sql
-- Create external stage
CREATE STAGE s3_stage
  URL = 's3://my-bucket/path/'
  CREDENTIALS = (AWS_KEY_ID = 'xxx' AWS_SECRET_KEY = 'yyy')
  FILE_FORMAT = (TYPE = 'PARQUET');

-- Load from external stage
COPY INTO my_table
  FROM @s3_stage/data.parquet
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

---

## Query Optimization

### Clustering Keys

For large tables (multi-TB), define clustering keys:

```sql
-- Add clustering key
ALTER TABLE large_table CLUSTER BY (date, region);

-- Check clustering quality
SELECT SYSTEM$CLUSTERING_INFORMATION('large_table', '(date, region)');

-- Automatic clustering (Enterprise)
ALTER TABLE large_table RESUME RECLUSTER;
```

### Query Profile Analysis

```sql
-- Enable query profiling
ALTER SESSION SET USE_CACHED_RESULT = FALSE;

-- Run query
SELECT ... ;

-- View query history
SELECT query_id, query_text, execution_time, warehouse_size, credits_used
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
ORDER BY credits_used DESC;
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Using ACCOUNTADMIN for queries | God-mode for routine work | Use SYSADMIN or functional role |
| No auto-suspend on warehouses | Wastes credits continuously | Set auto-suspend 1-10 minutes |
| GRANT ALL ON ACCOUNT | Excessive privileges | Grant specific privileges only |
| Single large warehouse | Inefficient, costly | Separate warehouses by workload |
| No role hierarchy | Privilege sprawl | Use role hierarchy (SYSADMIN → custom) |
| Cloning entire prod to dev daily | Unnecessary storage cost | Clone only needed schemas |
| Multi-cluster without monitoring | Auto-scaling costs spiral | Monitor before enabling |
| No clustering on large tables | Slow queries, high credits | Cluster by frequently filtered columns |
| SELECT * in production ETL | Transfers unnecessary data | Select only needed columns |
| No query timeout settings | Runaway queries waste credits | Set statement timeout |

---

## Quick Reference

### Daily Commands (Safe)

```sql
-- Check context
SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_DATABASE(), CURRENT_WAREHOUSE();

-- List objects
SHOW DATABASES;
SHOW TABLES IN SCHEMA mydb.myschema;
SHOW WAREHOUSES;

-- Describe objects
DESCRIBE TABLE mydb.myschema.mytable;
DESCRIBE WAREHOUSE my_wh;

-- Query data
SELECT * FROM mytable LIMIT 10;
SELECT COUNT(*) FROM mytable;

-- Check grants
SHOW GRANTS TO ROLE my_role;
SHOW GRANTS ON TABLE mytable;

-- Monitor warehouse
SHOW PARAMETERS FOR WAREHOUSE my_wh;
```

### DDL Commands (Review Required)

```sql
-- Create database
CREATE DATABASE myapp_dev;

-- Create warehouse
CREATE WAREHOUSE dev_wh
  WITH WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;

-- Create schema
CREATE SCHEMA myapp_dev.analytics;

-- Create table
CREATE TABLE myapp_dev.analytics.metrics (
  id NUMBER AUTOINCREMENT,
  timestamp TIMESTAMP_NTZ,
  metric_name VARCHAR(100),
  value FLOAT
);

-- Grant privileges
GRANT USAGE ON DATABASE myapp_dev TO ROLE dev_user;
GRANT USAGE ON SCHEMA myapp_dev.analytics TO ROLE dev_user;
GRANT SELECT ON ALL TABLES IN SCHEMA myapp_dev.analytics TO ROLE dev_user;
```

### DML Commands (Confirmation for Production)

```sql
-- Insert data
INSERT INTO mytable VALUES (1, 'value');

-- Update data
UPDATE mytable SET column = 'new_value' WHERE id = 1;

-- Delete data
DELETE FROM mytable WHERE id = 1;

-- Merge (upsert)
MERGE INTO target USING source
  ON target.id = source.id
  WHEN MATCHED THEN UPDATE SET target.value = source.value
  WHEN NOT MATCHED THEN INSERT (id, value) VALUES (source.id, source.value);
```

### Dangerous Commands (Confirmation Required)

```sql
-- Drop objects (REQUIRES CONFIRMATION)
DROP DATABASE myapp_dev;
DROP TABLE mytable;
TRUNCATE TABLE mytable;

-- Grant admin roles (REQUIRES CONFIRMATION)
GRANT ACCOUNTADMIN TO USER admin_user;
GRANT SYSADMIN TO ROLE custom_admin;

-- Large warehouse (REQUIRES CONFIRMATION)
CREATE WAREHOUSE prod_wh WITH WAREHOUSE_SIZE = 'XLARGE';
```

---

## Monitoring and Auditing

### Credit Consumption

```sql
-- Warehouse credit usage (last 30 days)
SELECT
  warehouse_name,
  SUM(credits_used) AS total_credits,
  SUM(credits_used) * 0.40 AS estimated_cost_usd
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY total_credits DESC;
```

### Query Monitoring

```sql
-- Expensive queries (last 7 days)
SELECT
  query_id,
  user_name,
  warehouse_name,
  execution_time / 1000 AS execution_seconds,
  credits_used_cloud_services,
  query_text
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND credits_used_cloud_services > 1
ORDER BY credits_used_cloud_services DESC
LIMIT 20;
```

### Failed Login Attempts

```sql
-- Failed logins (last 24 hours)
SELECT
  event_timestamp,
  user_name,
  client_ip,
  error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND is_success = 'NO'
ORDER BY event_timestamp DESC;
```

### Grant Audit

```sql
-- All grants to a user
SHOW GRANTS TO USER my_user;

-- All grants on a table
SHOW GRANTS ON TABLE mydb.myschema.mytable;

-- All users with ACCOUNTADMIN
SHOW GRANTS OF ROLE ACCOUNTADMIN;
```

---

## Pre-Execution Checklist

Before ANY Snowflake operation:

1. **Verify context**
   ```sql
   SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_DATABASE(), CURRENT_WAREHOUSE();
   ```

2. **Verify environment** (from database/warehouse name)
   - Is this dev, staging, or prod?
   - Are you sure?

3. **Verify role**
   - Using appropriate role for task?
   - Not using ACCOUNTADMIN unless necessary?

4. **Estimate cost impact** (for warehouse operations)
   - What size warehouse?
   - How long will query run?

5. **Check for dependencies** (for DROP operations)
   - What depends on this object?
   - Will deletion cascade?

---

## Audit Trail

All Snowflake operations are logged to:
```
~/.claude/logs/snowflake-operations.log  # Detailed log
~/.claude/logs/snowflake-audit.log       # Structured audit trail
```

### Audit Log Format

```
[2026-02-14 10:30:45] ACTION=ALLOW COMMAND="SELECT * FROM table" RESULT=read-only USER=donmccarty DATABASE=myapp_dev
[2026-02-14 10:31:02] ACTION=BLOCK COMMAND="DROP DATABASE prod" RESULT=security-violation USER=donmccarty
```

---

## References

- [Snowflake Documentation](https://docs.snowflake.com/)
- [SnowPro Core Certification](https://www.snowflake.com/certifications/)
- [Cost Optimization Guide](https://docs.snowflake.com/en/user-guide/cost-understanding)
- [Security Best Practices](https://docs.snowflake.com/en/user-guide/security)
- [Query Optimization](https://docs.snowflake.com/en/user-guide/query-optimization)
