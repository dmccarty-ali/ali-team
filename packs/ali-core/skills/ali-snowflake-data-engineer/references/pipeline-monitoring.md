# ali-snowflake-data-engineer - Pipeline Monitoring Reference

## Stream Health Monitoring

```sql
-- Check stream staleness
SHOW STREAMS;

-- Get stream metadata
SELECT SYSTEM$STREAM_GET_TABLE_TIMESTAMP('my_stream');

-- Monitor stream consumption
SELECT
    name,
    stale,
    stale_after,
    DATEDIFF('minute', stale_after, CURRENT_TIMESTAMP()) as minutes_until_stale
FROM TABLE(INFORMATION_SCHEMA.STREAMS())
WHERE stale = FALSE;
```

## Task Monitoring

```sql
-- Task run history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('day', -1, CURRENT_TIMESTAMP()),
    TASK_NAME => 'my_task'
))
ORDER BY scheduled_time DESC;

-- Failed tasks
SELECT
    name,
    scheduled_time,
    state,
    error_message
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())
WHERE state = 'FAILED'
  AND scheduled_time > DATEADD('day', -7, CURRENT_TIMESTAMP());

-- Task graph dependencies
SHOW TASKS;
```

## Alerts

```sql
-- Create alert for failed tasks
CREATE ALERT task_failure_alert
  WAREHOUSE = alert_wh
  SCHEDULE = '5 MINUTE'
  IF (EXISTS (
    SELECT 1
    FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())
    WHERE state = 'FAILED'
      AND scheduled_time > DATEADD('minute', -5, CURRENT_TIMESTAMP())
  ))
  THEN
    CALL SYSTEM$SEND_EMAIL(
      'ops@company.com',
      'Task Failure Alert',
      'A task has failed in the last 5 minutes'
    );

-- Enable alert
ALTER ALERT task_failure_alert RESUME;
```
