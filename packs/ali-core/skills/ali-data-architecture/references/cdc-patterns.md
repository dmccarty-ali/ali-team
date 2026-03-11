# ali-data-architecture - Change Data Capture Patterns Reference

Complete CDC implementations for tracking changes. See lines 638-709 of original SKILL.md.

---

## Custom CDC Pattern

### Source Table with CDC Columns

```sql
CREATE TABLE source.customers (
    customer_id     VARCHAR(50) PRIMARY KEY,
    name            VARCHAR(200),
    email           VARCHAR(255),
    -- CDC tracking
    created_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
    is_deleted      BOOLEAN DEFAULT FALSE,
    deleted_at      TIMESTAMP_NTZ
);
```

### Extract Changes

```sql
CREATE OR REPLACE PROCEDURE extract_customer_changes()
RETURNS STRING
AS
$$
DECLARE
    last_extract TIMESTAMP_NTZ;
BEGIN
    SELECT MAX(extract_timestamp) INTO last_extract
    FROM SYS_CONTROL.EXTRACT_LOG
    WHERE table_name = 'customers' AND status = 'success';

    INSERT INTO staging.customer_changes
    SELECT
        customer_id,
        name,
        email,
        CASE
            WHEN is_deleted THEN 'D'
            WHEN created_at > :last_extract THEN 'I'
            ELSE 'U'
        END AS operation,
        GREATEST(created_at, updated_at, COALESCE(deleted_at, '1900-01-01')) AS change_timestamp
    FROM source.customers
    WHERE updated_at > :last_extract
       OR created_at > :last_extract
       OR deleted_at > :last_extract;

    RETURN 'Extracted ' || SQLROWCOUNT || ' changes';
END;
$$;
```

---

## Snowflake Streams (Native CDC)

```sql
-- Create stream on source table
CREATE STREAM customer_stream ON TABLE source.customers;

-- Process changes
MERGE INTO target.customers tgt
USING (
    SELECT
        customer_id,
        name,
        email,
        METADATA$ACTION AS action,      -- INSERT, DELETE
        METADATA$ISUPDATE AS is_update, -- TRUE if part of UPDATE
        METADATA$ROW_ID AS row_id
    FROM customer_stream
) src
ON tgt.customer_id = src.customer_id

WHEN MATCHED AND src.action = 'DELETE' THEN
    DELETE

WHEN MATCHED AND src.action = 'INSERT' THEN
    UPDATE SET
        tgt.name = src.name,
        tgt.email = src.email,
        tgt.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED AND src.action = 'INSERT' THEN
    INSERT (customer_id, name, email, created_at)
    VALUES (src.customer_id, src.name, src.email, CURRENT_TIMESTAMP);

-- Stream is automatically advanced after successful DML
```

---

## Summary

- **Custom CDC**: Add created_at, updated_at, is_deleted columns, extract based on timestamps
- **Snowflake Streams**: Native CDC with METADATA$ columns, automatically tracks changes
- **Change Table**: Separate audit table with before/after values (see original implementation)
