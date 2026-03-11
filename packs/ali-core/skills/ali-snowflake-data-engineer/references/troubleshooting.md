# ali-snowflake-data-engineer - Troubleshooting Reference

## Common COPY INTO Errors

```sql
-- Validate before loading
COPY INTO my_table
FROM @my_stage
VALIDATION_MODE = 'RETURN_ERRORS';

-- Check load history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'my_table',
    START_TIME => DATEADD('hour', -24, CURRENT_TIMESTAMP())
))
WHERE STATUS != 'Loaded';

-- Common error patterns:
-- "Number of columns in file does not match"
--   → Check FILE_FORMAT ERROR_ON_COLUMN_COUNT_MISMATCH
-- "Numeric value 'abc' is not recognized"
--   → Data type mismatch, check source data
-- "Field delimiter not found"
--   → Wrong FIELD_DELIMITER in file format
```

## Snowpipe Troubleshooting

```sql
-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('my_pipe');

-- Force refresh (reprocess notifications)
ALTER PIPE my_pipe REFRESH;

-- Check pipe history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
    DATE_RANGE_START => DATEADD('day', -7, CURRENT_TIMESTAMP())
))
WHERE PIPE_NAME = 'MY_PIPE';

-- Check for stale files (not being picked up)
SELECT * FROM TABLE(INFORMATION_SCHEMA.VALIDATE_PIPE_LOAD(
    PIPE_NAME => 'my_pipe',
    START_TIME => DATEADD('hour', -1, CURRENT_TIMESTAMP())
));
```

## Query Profile Deep Dive - SQL Examples

```sql
-- Top 10 slowest queries (last 24h)
SELECT
    query_id,
    query_text,
    total_elapsed_time/1000 as seconds,
    bytes_scanned/1e9 as gb_scanned,
    partitions_scanned,
    partitions_total,
    bytes_spilled_to_local_storage/1e9 as gb_spilled_local,
    bytes_spilled_to_remote_storage/1e9 as gb_spilled_remote
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD('hour', -24, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
ORDER BY total_elapsed_time DESC
LIMIT 10;

-- Queries with poor partition pruning
SELECT
    query_id,
    query_text,
    partitions_scanned,
    partitions_total,
    ROUND(partitions_scanned/NULLIF(partitions_total, 0) * 100, 2) as pct_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD('hour', -24, CURRENT_TIMESTAMP())
  AND partitions_total > 1000
  AND partitions_scanned/NULLIF(partitions_total, 0) > 0.5
ORDER BY partitions_scanned DESC;
```
