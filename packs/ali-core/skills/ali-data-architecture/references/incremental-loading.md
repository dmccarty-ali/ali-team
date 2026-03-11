# ali-data-architecture - Incremental Loading Reference

Complete patterns for watermark, MERGE, and CDC-based incremental loads.

---

## Watermark Pattern

Track high-water mark for incremental extracts.

### Watermark Control Table

```sql
CREATE TABLE SYS_CONTROL.LOAD_WATERMARKS (
    table_name          VARCHAR(100) PRIMARY KEY,
    watermark_column    VARCHAR(100) NOT NULL,  -- Column used for incremental (updated_at, id, etc.)
    last_load_value     VARCHAR(200),           -- Last extracted value
    last_load_timestamp TIMESTAMP_NTZ,          -- When last load completed
    load_status         VARCHAR(20),            -- SUCCESS, FAILED, IN_PROGRESS
    load_batch_id       VARCHAR(100)
);

-- Initialize watermark for a table
INSERT INTO SYS_CONTROL.LOAD_WATERMARKS (
    table_name,
    watermark_column,
    last_load_value,
    last_load_timestamp,
    load_status
) VALUES (
    'source_system.customers',
    'updated_at',
    '1900-01-01 00:00:00',  -- Start from beginning
    '1900-01-01 00:00:00',
    'SUCCESS'
);
```

### Extract Using Watermark

```sql
-- Get current watermark
SET last_watermark = (
    SELECT last_load_value
    FROM SYS_CONTROL.LOAD_WATERMARKS
    WHERE table_name = 'source_system.customers'
);

-- Extract incremental data
CREATE TEMP TABLE staging_customers AS
SELECT *
FROM source_system.customers
WHERE updated_at > $last_watermark
ORDER BY updated_at;  -- Important for deterministic batching

-- Update watermark after successful load
UPDATE SYS_CONTROL.LOAD_WATERMARKS
SET
    last_load_value = (SELECT MAX(updated_at)::VARCHAR FROM staging_customers),
    last_load_timestamp = CURRENT_TIMESTAMP,
    load_status = 'SUCCESS',
    load_batch_id = $current_batch_id
WHERE table_name = 'source_system.customers';
```

### Watermark with Lookback Window

```sql
-- Extract with lookback (catch late-arriving updates)
SET last_watermark = (
    SELECT last_load_value::TIMESTAMP_NTZ - INTERVAL '1 HOUR'  -- 1-hour lookback
    FROM SYS_CONTROL.LOAD_WATERMARKS
    WHERE table_name = 'source_system.orders'
);

SELECT *
FROM source_system.orders
WHERE updated_at > $last_watermark;
```

---

## MERGE Pattern (Upsert)

Insert new records, update existing records.

### Basic MERGE

```sql
MERGE INTO STG_MEDEDI.CLAIM tgt
USING staging_claims src
ON tgt.claim_id = src.claim_id

WHEN MATCHED AND src.updated_at > tgt.load_timestamp THEN
    UPDATE SET
        tgt.patient_id = src.patient_id,
        tgt.provider_npi = src.provider_npi,
        tgt.total_charge = src.total_charge,
        tgt.status = src.status,
        tgt.load_timestamp = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (claim_id, patient_id, provider_npi, total_charge, status, load_timestamp)
    VALUES (src.claim_id, src.patient_id, src.provider_npi, src.total_charge, src.status, CURRENT_TIMESTAMP);
```

### MERGE with Deduplication

```sql
-- Deduplicate source before MERGE (keep latest version)
MERGE INTO STG_MEDEDI.CLAIM tgt
USING (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY claim_id
            ORDER BY updated_at DESC  -- Latest version wins
        ) AS rn
    FROM staging_claims
) src
ON tgt.claim_id = src.claim_id AND src.rn = 1

WHEN MATCHED THEN
    UPDATE SET
        tgt.total_charge = src.total_charge,
        tgt.status = src.status,
        tgt.load_timestamp = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (claim_id, total_charge, status, load_timestamp)
    VALUES (src.claim_id, src.total_charge, src.status, CURRENT_TIMESTAMP);
```

### MERGE with Conditional Updates

```sql
-- Only update if values actually changed
MERGE INTO STG_MEDEDI.CLAIM tgt
USING staging_claims src
ON tgt.claim_id = src.claim_id

WHEN MATCHED AND (
    tgt.total_charge != src.total_charge OR
    tgt.status != src.status OR
    tgt.provider_npi != src.provider_npi
) THEN
    UPDATE SET
        tgt.total_charge = src.total_charge,
        tgt.status = src.status,
        tgt.provider_npi = src.provider_npi,
        tgt.load_timestamp = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (claim_id, total_charge, status, provider_npi, load_timestamp)
    VALUES (src.claim_id, src.total_charge, src.status, src.provider_npi, CURRENT_TIMESTAMP);
```

### MERGE with Soft Deletes

```sql
-- Mark records as deleted if no longer in source
MERGE INTO STG_MEDEDI.CLAIM tgt
USING (
    SELECT claim_id FROM staging_claims
) src
ON tgt.claim_id = src.claim_id

WHEN MATCHED THEN
    UPDATE SET
        tgt.is_deleted = FALSE,
        tgt.load_timestamp = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (claim_id, load_timestamp)
    VALUES (src.claim_id, CURRENT_TIMESTAMP);

-- Separately mark records not in current batch as deleted
UPDATE STG_MEDEDI.CLAIM
SET
    is_deleted = TRUE,
    deleted_at = CURRENT_TIMESTAMP
WHERE claim_id NOT IN (SELECT claim_id FROM staging_claims)
  AND is_deleted = FALSE;
```

---

## DELETE + INSERT Pattern

Simple alternative to MERGE for small batches.

### Partition-Based Delete/Insert

```sql
-- Delete partition
BEGIN TRANSACTION;

DELETE FROM STG_MEDEDI.CLAIM
WHERE service_date BETWEEN '2026-01-01' AND '2026-01-31';

-- Insert partition
INSERT INTO STG_MEDEDI.CLAIM
SELECT * FROM staging_claims
WHERE service_date BETWEEN '2026-01-01' AND '2026-01-31';

COMMIT;
```

### Batch-Based Delete/Insert

```sql
-- Delete by batch ID
DELETE FROM STG_MEDEDI.CLAIM
WHERE load_batch_id = $current_batch_id;

-- Insert new batch
INSERT INTO STG_MEDEDI.CLAIM
SELECT
    *,
    $current_batch_id AS load_batch_id
FROM staging_claims;
```

---

## Hash-Based Change Detection

Avoid updating rows that haven't changed.

### Row Hash Calculation

```sql
-- Add row_hash column to table
ALTER TABLE STG_MEDEDI.CLAIM ADD COLUMN row_hash VARCHAR(64);

-- Calculate hash of row content
UPDATE STG_MEDEDI.CLAIM
SET row_hash = MD5(
    COALESCE(claim_id, '') || '|' ||
    COALESCE(patient_id, '') || '|' ||
    COALESCE(provider_npi, '') || '|' ||
    COALESCE(total_charge::VARCHAR, '') || '|' ||
    COALESCE(status, '')
);
```

### MERGE with Hash Comparison

```sql
-- Only update if row hash changed
MERGE INTO STG_MEDEDI.CLAIM tgt
USING (
    SELECT
        *,
        MD5(
            COALESCE(claim_id, '') || '|' ||
            COALESCE(patient_id, '') || '|' ||
            COALESCE(provider_npi, '') || '|' ||
            COALESCE(total_charge::VARCHAR, '') || '|' ||
            COALESCE(status, '')
        ) AS row_hash
    FROM staging_claims
) src
ON tgt.claim_id = src.claim_id

WHEN MATCHED AND tgt.row_hash != src.row_hash THEN
    UPDATE SET
        tgt.patient_id = src.patient_id,
        tgt.provider_npi = src.provider_npi,
        tgt.total_charge = src.total_charge,
        tgt.status = src.status,
        tgt.row_hash = src.row_hash,
        tgt.load_timestamp = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (claim_id, patient_id, provider_npi, total_charge, status, row_hash, load_timestamp)
    VALUES (src.claim_id, src.patient_id, src.provider_npi, src.total_charge, src.status, src.row_hash, CURRENT_TIMESTAMP);
```

---

## Incremental Loading with Snowflake Streams

Snowflake's native CDC using streams.

### Create Stream

```sql
-- Create stream on source table
CREATE STREAM customer_stream ON TABLE source.customers;

-- Stream tracks INSERT, UPDATE, DELETE automatically
```

### Process Stream

```sql
-- Process changes from stream
MERGE INTO target.customers tgt
USING (
    SELECT
        customer_id,
        first_name,
        last_name,
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
        tgt.first_name = src.first_name,
        tgt.last_name = src.last_name,
        tgt.email = src.email,
        tgt.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED AND src.action = 'INSERT' THEN
    INSERT (customer_id, first_name, last_name, email, created_at)
    VALUES (src.customer_id, src.first_name, src.last_name, src.email, CURRENT_TIMESTAMP);

-- Stream is automatically advanced after successful DML
```

### Stream Offset (Replay)

```sql
-- Clone stream at specific point
CREATE STREAM customer_stream_backup CLONE customer_stream AT (TIMESTAMP => '2026-01-15 10:00:00'::TIMESTAMP_NTZ);

-- Process from backup stream
SELECT * FROM customer_stream_backup;
```

---

## Idempotency Considerations

### Deterministic Batch IDs

```sql
-- Use date-based batch ID for idempotency
SET batch_id = TO_CHAR(CURRENT_TIMESTAMP, 'YYYYMMDDHH24MISS');

-- If pipeline re-runs, same batch ID ensures same behavior
INSERT INTO STG_MEDEDI.CLAIM
SELECT
    *,
    $batch_id AS load_batch_id
FROM staging_claims
WHERE NOT EXISTS (
    SELECT 1 FROM STG_MEDEDI.CLAIM
    WHERE load_batch_id = $batch_id
);
```

### Tombstone Pattern (Soft Deletes)

```sql
-- Don't DELETE, mark as deleted
UPDATE STG_MEDEDI.CLAIM
SET
    is_deleted = TRUE,
    deleted_at = CURRENT_TIMESTAMP
WHERE claim_id IN (
    SELECT claim_id FROM deleted_claims_staging
);

-- Downstream queries filter out deleted
SELECT *
FROM STG_MEDEDI.CLAIM
WHERE is_deleted = FALSE;
```

---

## Performance Optimization

### Partition Pruning

```sql
-- Partition table by load date
CREATE TABLE STG_MEDEDI.CLAIM (
    ...
    load_date DATE NOT NULL
)
PARTITION BY RANGE (load_date);

-- Query benefits from partition pruning
SELECT *
FROM STG_MEDEDI.CLAIM
WHERE load_date = CURRENT_DATE;  -- Only scans one partition
```

### Clustering

```sql
-- Cluster by frequently filtered columns
ALTER TABLE STG_MEDEDI.CLAIM
CLUSTER BY (service_date, provider_npi);

-- Incremental queries benefit from clustering
SELECT *
FROM STG_MEDEDI.CLAIM
WHERE service_date >= CURRENT_DATE - 7
  AND provider_npi = '1234567890';
```

### Parallel MERGE

```sql
-- Partition source data for parallel processing
CREATE TEMP TABLE staging_claims_part1 AS
SELECT * FROM staging_claims WHERE MOD(HASH(claim_id), 4) = 0;

CREATE TEMP TABLE staging_claims_part2 AS
SELECT * FROM staging_claims WHERE MOD(HASH(claim_id), 4) = 1;

-- Run MERGE statements in parallel (separate sessions)
MERGE INTO STG_MEDEDI.CLAIM ... FROM staging_claims_part1 ...;  -- Session 1
MERGE INTO STG_MEDEDI.CLAIM ... FROM staging_claims_part2 ...;  -- Session 2
```

---

## Error Handling

### Transaction Rollback

```sql
BEGIN TRANSACTION;

-- Set watermark to IN_PROGRESS
UPDATE SYS_CONTROL.LOAD_WATERMARKS
SET load_status = 'IN_PROGRESS'
WHERE table_name = 'source.customers';

-- Attempt load
BEGIN TRY
    MERGE INTO target.customers ... ;

    -- Update watermark to SUCCESS
    UPDATE SYS_CONTROL.LOAD_WATERMARKS
    SET
        last_load_value = $new_watermark,
        last_load_timestamp = CURRENT_TIMESTAMP,
        load_status = 'SUCCESS'
    WHERE table_name = 'source.customers';

    COMMIT;
END TRY
BEGIN CATCH
    -- Rollback on error
    ROLLBACK;

    -- Update watermark to FAILED
    UPDATE SYS_CONTROL.LOAD_WATERMARKS
    SET load_status = 'FAILED'
    WHERE table_name = 'source.customers';

    RAISE;
END CATCH;
```

---

## Summary

- **Watermark**: Track last extracted value (timestamp, ID)
- **MERGE**: Upsert pattern (insert new, update existing)
- **Hash-based**: Avoid updates when content unchanged
- **Streams**: Snowflake native CDC
- **Idempotency**: Deterministic batch IDs, tombstone deletes
- **Performance**: Partitioning, clustering, parallel processing
