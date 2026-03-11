# ali-snowflake-data-engineer - QUALIFY Advanced Patterns Reference

## Microsecond Precision for Deterministic Ordering

```sql
-- Use TIMESTAMP_NTZ(6) for microsecond precision
-- Ensures deterministic ordering when multiple updates in same second
CREATE TABLE file_status (
    file_key        VARCHAR(500),
    status          VARCHAR(50),
    created_at      TIMESTAMP_NTZ(6)  -- 6 digits = microseconds
);

-- Parser provides microsecond timestamp
INSERT INTO file_status VALUES (
    'path/to/file.edi',
    'PROCESSED',
    '2024-01-15 10:30:45.123456'  -- Microsecond precision
);

-- Latest status per file (deterministic even with rapid updates)
CREATE VIEW FILES_STATUS_LATEST AS
SELECT file_key, status, created_at
FROM file_status
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY file_key
    ORDER BY created_at DESC  -- Microseconds ensure unique ordering
) = 1;
```

## Multi-Column Deduplication

```sql
-- Latest record per composite key
SELECT *
FROM ingestion_events
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY batch_id, parser_seq, table_name
    ORDER BY event_timestamp DESC
) = 1;
```
