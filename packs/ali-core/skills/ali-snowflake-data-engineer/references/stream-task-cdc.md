# ali-snowflake-data-engineer - Stream + Task CDC Pattern Reference

## Complete CDC Implementation

```sql
-- Source table
CREATE TABLE source_data (...);

-- Stream to capture changes
CREATE STREAM source_changes ON TABLE source_data;

-- Staging table for processing
CREATE TABLE staged_changes (...);

-- Task 1: Stage changes
CREATE TASK stage_changes
  WAREHOUSE = etl_wh
  SCHEDULE = '1 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('source_changes')
AS
INSERT INTO staged_changes
SELECT *, METADATA$ACTION, METADATA$ISUPDATE, CURRENT_TIMESTAMP()
FROM source_changes;

-- Task 2: Apply to target (depends on task 1)
CREATE TASK apply_changes
  WAREHOUSE = etl_wh
  AFTER stage_changes
AS
MERGE INTO target_table t
USING staged_changes s ON t.id = s.id
WHEN MATCHED AND s.action = 'DELETE' THEN DELETE
WHEN MATCHED AND s.action = 'INSERT' THEN UPDATE SET ...
WHEN NOT MATCHED AND s.action = 'INSERT' THEN INSERT ...;

-- Task 3: Cleanup
CREATE TASK cleanup_staged
  WAREHOUSE = etl_wh
  AFTER apply_changes
AS
DELETE FROM staged_changes WHERE processed_at < DATEADD('hour', -24, CURRENT_TIMESTAMP());

-- Enable task tree (start from root)
ALTER TASK cleanup_staged RESUME;
ALTER TASK apply_changes RESUME;
ALTER TASK stage_changes RESUME;
```
