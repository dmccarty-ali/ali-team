# ali-data-architecture - Slowly Changing Dimension Patterns Reference

Complete SQL implementations for all three SCD types.

---

## SCD Type 1: Overwrite (No History)

**Use when:** Changes are corrections OR history doesn't matter for the attribute.

**Example:** Fixing a typo in provider name, updating phone numbers.

```sql
-- Simple UPDATE - loses history
UPDATE DIM_PROVIDER
SET provider_name = 'Dr. Jane Smith'  -- Was 'Dr. Jane Smoth' (typo)
WHERE provider_npi = '1234567890';

-- MERGE version (upsert pattern)
MERGE INTO DIM_PROVIDER tgt
USING staging_providers src
ON tgt.provider_npi = src.provider_npi
WHEN MATCHED THEN UPDATE SET
    tgt.provider_name = src.provider_name,
    tgt.specialty = src.specialty,
    tgt.address_city = src.address_city,
    tgt.address_state = src.address_state,
    tgt.phone = src.phone,
    tgt.load_timestamp = CURRENT_TIMESTAMP
WHEN NOT MATCHED THEN INSERT (
    provider_sk,
    provider_npi,
    provider_name,
    specialty,
    load_timestamp
) VALUES (
    seq_provider_sk.NEXTVAL,
    src.provider_npi,
    src.provider_name,
    src.specialty,
    CURRENT_TIMESTAMP
);
```

**Characteristics:**
- Simple and fast
- No history preserved
- Smallest storage footprint
- Good for rarely-queried attributes

---

## SCD Type 2: Add New Row (Full History)

**Use when:** History matters for the attribute (e.g., provider specialty changes, patient address changes).

**Pattern:** Close current row (set valid_to, is_current=FALSE), insert new row with new values.

```sql
-- Full Type 2 implementation with MERGE
MERGE INTO DIM_PROVIDER tgt
USING (
    -- Identify records that changed
    SELECT
        src.*,
        tgt.provider_sk AS existing_sk,
        tgt.valid_from AS existing_valid_from
    FROM staging_providers src
    LEFT JOIN DIM_PROVIDER tgt
        ON tgt.provider_npi = src.provider_npi
        AND tgt.is_current = TRUE
    WHERE tgt.provider_sk IS NULL  -- New provider
       OR tgt.specialty != src.specialty  -- Specialty changed
       OR tgt.address_city != src.address_city  -- Address changed
) src
ON tgt.provider_sk = src.existing_sk AND tgt.is_current = TRUE

-- Close existing current row
WHEN MATCHED THEN UPDATE SET
    tgt.valid_to = CURRENT_DATE - 1,
    tgt.is_current = FALSE,
    tgt.load_timestamp = CURRENT_TIMESTAMP

-- Insert new version
WHEN NOT MATCHED THEN INSERT (
    provider_sk,
    provider_npi,
    provider_name,
    specialty,
    address_city,
    address_state,
    valid_from,
    valid_to,
    is_current,
    load_timestamp
) VALUES (
    COALESCE(src.existing_sk, seq_provider_sk.NEXTVAL),  -- Reuse SK if exists
    src.provider_npi,
    src.provider_name,
    src.specialty,
    src.address_city,
    src.address_state,
    CURRENT_DATE,
    '9999-12-31',
    TRUE,
    CURRENT_TIMESTAMP
);

-- Separate INSERT for new rows from closed rows
-- (Since MERGE can't INSERT multiple times for same key)
INSERT INTO DIM_PROVIDER (
    provider_sk,
    provider_npi,
    provider_name,
    specialty,
    valid_from,
    valid_to,
    is_current,
    load_timestamp
)
SELECT
    seq_provider_sk.NEXTVAL,
    src.provider_npi,
    src.provider_name,
    src.specialty,
    CURRENT_DATE,
    '9999-12-31',
    TRUE,
    CURRENT_TIMESTAMP
FROM staging_providers src
WHERE EXISTS (
    SELECT 1 FROM DIM_PROVIDER tgt
    WHERE tgt.provider_npi = src.provider_npi
    AND tgt.is_current = FALSE
    AND tgt.valid_to = CURRENT_DATE - 1
);
```

**Alternative: Two-statement pattern (clearer logic)**

```sql
-- Step 1: Close current rows that changed
UPDATE DIM_PROVIDER tgt
SET
    tgt.valid_to = CURRENT_DATE - 1,
    tgt.is_current = FALSE,
    tgt.load_timestamp = CURRENT_TIMESTAMP
WHERE tgt.is_current = TRUE
AND EXISTS (
    SELECT 1 FROM staging_providers src
    WHERE src.provider_npi = tgt.provider_npi
    AND (
        src.specialty != tgt.specialty
        OR src.address_city != tgt.address_city
    )
);

-- Step 2: Insert new versions
INSERT INTO DIM_PROVIDER (
    provider_sk,
    provider_npi,
    provider_name,
    specialty,
    address_city,
    address_state,
    valid_from,
    valid_to,
    is_current,
    load_timestamp
)
SELECT
    seq_provider_sk.NEXTVAL,
    src.provider_npi,
    src.provider_name,
    src.specialty,
    src.address_city,
    src.address_state,
    CURRENT_DATE,
    '9999-12-31',
    TRUE,
    CURRENT_TIMESTAMP
FROM staging_providers src
WHERE NOT EXISTS (
    SELECT 1 FROM DIM_PROVIDER tgt
    WHERE tgt.provider_npi = src.provider_npi
    AND tgt.is_current = TRUE
    AND tgt.specialty = src.specialty  -- No change
    AND tgt.address_city = src.address_city
);
```

**Querying Type 2 dimensions:**

```sql
-- Get current record only
SELECT * FROM DIM_PROVIDER
WHERE provider_npi = '1234567890'
AND is_current = TRUE;

-- Get record as of specific date
SELECT * FROM DIM_PROVIDER
WHERE provider_npi = '1234567890'
AND valid_from <= '2025-06-01'
AND valid_to >= '2025-06-01';

-- Join fact to dimension (as of service date)
SELECT
    fact.claim_id,
    fact.service_date,
    dim.provider_name,
    dim.specialty  -- Specialty as it was on service date
FROM FACT_CLAIM fact
JOIN DIM_PROVIDER dim
    ON fact.provider_npi = dim.provider_npi
    AND fact.service_date BETWEEN dim.valid_from AND dim.valid_to;
```

**Characteristics:**
- Preserves full history
- Larger storage (one row per change)
- More complex queries (need date filtering)
- Essential for regulatory/audit requirements

---

## SCD Type 3: Add Column (Current + Previous)

**Use when:** Only need to track "before/after" (not full history), and there's a single previous value that matters.

**Example:** Track previous specialty when provider changes specialty (for transition analysis).

```sql
-- Add columns for previous values
ALTER TABLE DIM_PROVIDER ADD COLUMN previous_specialty VARCHAR(100);
ALTER TABLE DIM_PROVIDER ADD COLUMN specialty_change_date DATE;

-- Update logic
UPDATE DIM_PROVIDER SET
    previous_specialty = specialty,
    specialty = 'Cardiology',
    specialty_change_date = CURRENT_DATE
WHERE provider_npi = '1234567890';

-- Querying
SELECT
    provider_npi,
    provider_name,
    previous_specialty,
    specialty AS current_specialty,
    specialty_change_date
FROM DIM_PROVIDER
WHERE previous_specialty IS NOT NULL;  -- Providers who changed specialty
```

**Variant: Multiple previous values (Type 3 extended)**

```sql
-- Track current + last 2 specialties
ALTER TABLE DIM_PROVIDER ADD COLUMN previous_specialty_1 VARCHAR(100);
ALTER TABLE DIM_PROVIDER ADD COLUMN previous_specialty_2 VARCHAR(100);
ALTER TABLE DIM_PROVIDER ADD COLUMN specialty_change_date_1 DATE;
ALTER TABLE DIM_PROVIDER ADD COLUMN specialty_change_date_2 DATE;

-- Update with shift pattern
UPDATE DIM_PROVIDER SET
    previous_specialty_2 = previous_specialty_1,
    previous_specialty_1 = specialty,
    specialty = 'Cardiology',
    specialty_change_date_2 = specialty_change_date_1,
    specialty_change_date_1 = CURRENT_DATE
WHERE provider_npi = '1234567890';
```

**Characteristics:**
- Simple schema (no date filtering needed)
- Limited history (only current + N previous)
- Fast queries (no joins on dates)
- Good for "before/after" analysis

---

## Hybrid Approaches

### Type 1 + Type 2 (Different attributes, different strategies)

```sql
CREATE TABLE DIM_PROVIDER (
    provider_sk         INTEGER PRIMARY KEY,
    provider_npi        VARCHAR(10) NOT NULL,

    -- Type 1 attributes (overwrite, no history)
    provider_name       VARCHAR(200),           -- Corrections only
    phone               VARCHAR(20),            -- Current contact only
    email               VARCHAR(255),

    -- Type 2 attributes (track history)
    specialty           VARCHAR(100),           -- History matters
    address_city        VARCHAR(100),           -- History matters
    address_state       VARCHAR(2),
    network_status      VARCHAR(20),            -- In/out of network changes

    -- Type 2 columns
    valid_from          DATE NOT NULL,
    valid_to            DATE NOT NULL DEFAULT '9999-12-31',
    is_current          BOOLEAN NOT NULL DEFAULT TRUE,

    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP
);

-- Update logic: Type 1 changes don't create new rows
UPDATE DIM_PROVIDER
SET
    provider_name = src.provider_name,  -- Type 1: just update
    phone = src.phone,                  -- Type 1: just update
    load_timestamp = CURRENT_TIMESTAMP
WHERE provider_npi = src.provider_npi
AND is_current = TRUE
AND (specialty = src.specialty);  -- No Type 2 changes

-- Type 2 logic: Only when specialty/address/network changes
-- (Use Type 2 pattern from above)
```

### Mini-dimensions (Type 2 optimization for high-cardinality)

When dimension has millions of rows and frequently-changing attributes cause explosion:

```sql
-- Main dimension: Slowly-changing or static attributes only
CREATE TABLE DIM_CUSTOMER (
    customer_sk         INTEGER PRIMARY KEY,
    customer_id         VARCHAR(50) UNIQUE,
    customer_name       VARCHAR(200),
    date_of_birth       DATE,
    -- No address, no phone (too volatile)
    valid_from          DATE,
    valid_to            DATE,
    is_current          BOOLEAN
);

-- Mini-dimension: Frequently-changing attributes
CREATE TABLE DIM_CUSTOMER_DEMOGRAPHICS (
    demographics_sk     INTEGER PRIMARY KEY,
    income_band         VARCHAR(20),        -- Changes often
    credit_score_band   VARCHAR(20),        -- Changes often
    address_city        VARCHAR(100),       -- Changes often
    address_state       VARCHAR(2),
    valid_from          DATE,
    valid_to            DATE,
    is_current          BOOLEAN
);

-- Fact table: Join to both
CREATE TABLE FACT_TRANSACTION (
    transaction_sk      INTEGER PRIMARY KEY,
    customer_sk         INTEGER,            -- FK to main dimension
    demographics_sk     INTEGER,            -- FK to mini-dimension
    date_sk             INTEGER,
    amount              NUMBER(12,2)
);

-- Benefit: Mini-dimension has far fewer rows (unique combinations of bands/city/state)
-- Example: 1M customers * 100 address changes = 100M rows (Type 2)
--          vs. 1M customers + 50K unique demographic combinations = 1.05M rows (mini-dim)
```

---

## Comparison Table

| Aspect | Type 1 | Type 2 | Type 3 |
|--------|--------|--------|--------|
| **History preserved** | No | Full | Limited (1-2 previous) |
| **Storage** | Smallest | Largest | Medium |
| **Query complexity** | Simple | Complex (date filters) | Simple |
| **Use case** | Corrections, current-only | Audit, compliance | Before/after analysis |
| **Implementation** | UPDATE only | UPDATE + INSERT | ALTER + UPDATE |
| **Example** | Phone number | Provider specialty | Previous address |

---

## Best Practices

1. **Choose strategy per attribute, not per table**
   - Same dimension can have Type 1 and Type 2 attributes

2. **Always include is_current flag**
   - Simplifies queries (WHERE is_current = TRUE)
   - Faster than date range checks

3. **Use valid_to = '9999-12-31' convention**
   - Easier than NULL (avoids NULL handling in queries)
   - Explicit "this is current" semantics

4. **Index appropriately**
   - CREATE INDEX idx_provider_current ON DIM_PROVIDER (provider_npi, is_current);
   - CREATE INDEX idx_provider_dates ON DIM_PROVIDER (valid_from, valid_to);

5. **Consider mini-dimensions for high-cardinality + high-change**
   - Prevents dimension explosion
   - Reduces fact table size

6. **Document SCD strategy in metadata**
   - Make it clear which attributes are Type 1 vs Type 2
   - Include in data dictionary

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| **Forgetting to close old row** | Two current rows for same entity | Always UPDATE is_current=FALSE before INSERT |
| **Not checking for actual changes** | New row on every load (even if no change) | Compare old vs new values before INSERT |
| **Date overlap** | valid_to = same date as next valid_from | Use valid_to = next_valid_from - 1 day |
| **Surrogate key reuse** | New row gets new SK, breaks fact table joins | Reuse SK for same natural key (or use bridge table) |
| **Querying without date filter** | Get multiple rows for same entity | Always filter: is_current = TRUE OR date BETWEEN valid_from AND valid_to |
