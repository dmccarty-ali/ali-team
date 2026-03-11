# ali-snowflake-data-engineer - Query Acceleration Service Reference

## Enable Query Acceleration

```sql
-- Enable with scale factor
ALTER WAREHOUSE my_wh SET
  ENABLE_QUERY_ACCELERATION = TRUE
  QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8;  -- up to 8x warehouse size

-- Check eligibility
SELECT
    query_id,
    eligible_query_acceleration_time,
    upper_limit_scale_factor
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_ACCELERATION_ELIGIBLE
WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY eligible_query_acceleration_time DESC;

-- Cost monitoring
SELECT
    warehouse_name,
    SUM(credits_used) as acceleration_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_ACCELERATION_HISTORY
WHERE start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1;
```
