# ali-snowflake-data-engineer - Governance Reference

## Object Tagging Strategy

```sql
-- Create tag hierarchy
CREATE TAG data_domain ALLOWED_VALUES 'finance', 'hr', 'sales', 'marketing';
CREATE TAG pii_level ALLOWED_VALUES 'none', 'low', 'medium', 'high';
CREATE TAG data_owner;

-- Apply tags at different levels
ALTER DATABASE analytics SET TAG data_domain = 'finance';
ALTER SCHEMA analytics.reporting SET TAG data_owner = 'analytics-team@company.com';
ALTER TABLE customers MODIFY COLUMN ssn SET TAG pii_level = 'high';
ALTER TABLE customers MODIFY COLUMN email SET TAG pii_level = 'medium';

-- Query tagged objects
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TAG_REFERENCES_ALL_COLUMNS(
    'customers',
    'TABLE'
));

-- Find all high-PII columns
SELECT
    object_database,
    object_schema,
    object_name,
    column_name,
    tag_value
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE tag_name = 'PII_LEVEL'
  AND tag_value = 'high';
```

## Dynamic Data Masking

```sql
-- Conditional masking based on role
CREATE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('ADMIN', 'HR_FULL_ACCESS') THEN val
    WHEN CURRENT_ROLE() IN ('HR_LIMITED') THEN
      REGEXP_REPLACE(val, '(.{2}).*(@.*)', '\\1***\\2')
    ELSE '***@***.***'
  END;

-- Masking with context functions
CREATE MASKING POLICY salary_mask AS (val NUMBER) RETURNS NUMBER ->
  CASE
    WHEN IS_ROLE_IN_SESSION('HR_ADMIN') THEN val
    WHEN INVOKER_SHARE() IS NOT NULL THEN NULL  -- external share
    ELSE 0
  END;

-- Apply policy
ALTER TABLE employees MODIFY COLUMN salary SET MASKING POLICY salary_mask;

-- View active policies
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES
WHERE policy_kind = 'MASKING_POLICY';
```

## Row Access Policies

```sql
-- Multi-tenant row access
CREATE ROW ACCESS POLICY tenant_isolation
  AS (tenant_id VARCHAR) RETURNS BOOLEAN ->
    CURRENT_ROLE() = 'ADMIN'
    OR EXISTS (
      SELECT 1 FROM user_tenant_mapping
      WHERE user_name = CURRENT_USER()
        AND tenant = tenant_id
    );

-- Region-based access
CREATE ROW ACCESS POLICY region_access
  AS (region VARCHAR) RETURNS BOOLEAN ->
    region IN (
      SELECT allowed_region
      FROM role_region_access
      WHERE role_name = CURRENT_ROLE()
    );

-- Apply to table
ALTER TABLE transactions ADD ROW ACCESS POLICY tenant_isolation ON (tenant_id);
```

## Data Lineage

```sql
-- Access history (who read what)
SELECT
    query_start_time,
    user_name,
    direct_objects_accessed,
    base_objects_accessed,
    objects_modified
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE query_start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND ARRAY_CONTAINS('MY_DB.MY_SCHEMA.SENSITIVE_TABLE'::VARIANT, base_objects_accessed:objectName);

-- Column lineage (which columns were read)
SELECT
    query_id,
    objects_accessed.value:objectName::STRING as table_name,
    cols.value:columnName::STRING as column_name
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
     LATERAL FLATTEN(input => direct_objects_accessed) objects_accessed,
     LATERAL FLATTEN(input => objects_accessed.value:columns) cols
WHERE query_start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());
```

## Data Quality Monitoring

```sql
-- Built-in data metric functions
ALTER TABLE my_table SET DATA_METRIC_SCHEDULE = '1 HOUR';

-- Add data quality metrics
ALTER TABLE my_table ADD DATA METRIC FUNCTION
  SNOWFLAKE.CORE.NULL_COUNT ON (email);

ALTER TABLE my_table ADD DATA METRIC FUNCTION
  SNOWFLAKE.CORE.UNIQUE_COUNT ON (customer_id);

-- Query metric results
SELECT *
FROM TABLE(INFORMATION_SCHEMA.DATA_QUALITY_MONITORING_RESULTS(
    REF_ENTITY_NAME => 'MY_DB.MY_SCHEMA.MY_TABLE'
));

-- Custom quality checks with alerts
CREATE ALERT data_quality_alert
  WAREHOUSE = alert_wh
  SCHEDULE = 'USING CRON 0 * * * * UTC'
  IF (EXISTS (
    SELECT 1 FROM my_table
    WHERE email IS NULL OR email NOT LIKE '%@%.%'
    LIMIT 1
  ))
  THEN
    CALL notify_data_quality_issue();
```
