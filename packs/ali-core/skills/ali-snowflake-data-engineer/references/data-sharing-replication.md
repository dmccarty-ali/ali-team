# ali-snowflake-data-engineer - Data Sharing & Replication Reference

## Row-Level Filtering for Shares

```sql
-- Create secure view with row-level filter
CREATE SECURE VIEW shared_sales AS
SELECT * FROM sales
WHERE tenant_id = CURRENT_ACCOUNT();

-- Or use row access policy
CREATE ROW ACCESS POLICY tenant_filter AS (tenant_id VARCHAR) RETURNS BOOLEAN ->
  tenant_id = CURRENT_ACCOUNT();

-- Share the secure view
GRANT SELECT ON VIEW shared_sales TO SHARE my_share;
```

## Cross-Cloud Replication

```sql
-- Enable replication on database
ALTER DATABASE my_db ENABLE REPLICATION TO ACCOUNTS org.account_in_azure;

-- Create replica in target account
CREATE DATABASE my_db_replica
  AS REPLICA OF org.source_account.my_db;

-- Refresh replica
ALTER DATABASE my_db_replica REFRESH;
```

## Cross-Region Replication

```sql
-- Primary: Enable replication
ALTER DATABASE my_db ENABLE REPLICATION TO ACCOUNTS
  myorg.account_east,
  myorg.account_eu;

-- Secondary: Create replica
CREATE DATABASE my_db
  AS REPLICA OF myorg.account_west.my_db;

-- Set refresh schedule
ALTER DATABASE my_db SET
  DATA_RETENTION_TIME_IN_DAYS = 7;

-- Manual refresh
ALTER DATABASE my_db REFRESH;

-- Monitor replication lag
SELECT
    database_name,
    primary_snapshot_timestamp,
    secondary_snapshot_timestamp,
    DATEDIFF('minute', secondary_snapshot_timestamp, primary_snapshot_timestamp) as lag_minutes
FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUP_REFRESH_HISTORY
WHERE database_name = 'MY_DB'
ORDER BY start_time DESC
LIMIT 10;
```

## Failover/Failback

```sql
-- Promote secondary to primary (failover)
ALTER DATABASE my_db PRIMARY;

-- After recovery, fail back
-- On new secondary (old primary):
ALTER DATABASE my_db REFRESH;
```
