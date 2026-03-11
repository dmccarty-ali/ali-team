# ali-data-architecture - Medallion Architecture Reference

Complete schema examples for LAKE (Bronze), STG (Silver), and EDW (Gold) layers.

---

## LAKE Layer (Bronze) - Complete Schema

Raw data, minimal transformation. Schema matches source format.

```sql
-- Raw EDI 837I claim data
CREATE TABLE LAKE_MEDEDI.EDI_837I_RAW (
    file_id         VARCHAR(100),       -- Source file identifier (required for replay)
    record_type     VARCHAR(10),        -- Segment type (ISA, GS, ST, CLM, etc.)
    record_content  VARCHAR(16000),     -- Raw segment content (as received)
    line_number     INTEGER,            -- Position in file (sequential from 1)
    ingested_at     TIMESTAMP_NTZ,      -- When we received it (system time)
    source_path     VARCHAR(1000),      -- S3 path to source file (full path)

    -- Optional: metadata for troubleshooting
    file_size_bytes BIGINT,
    file_checksum   VARCHAR(64)         -- MD5 or SHA256 of original file
);

-- Key characteristics:
-- 1. Append-only (never UPDATE or DELETE)
-- 2. Preserves exact source format (VARCHAR for everything)
-- 3. Enables replay from raw data if downstream logic changes
-- 4. No transformations except minimal parsing (file → rows)
-- 5. Immutable - if bad data arrives, add correction, don't modify

-- Raw payment remittance (835)
CREATE TABLE LAKE_MEDEDI.EDI_835_RAW (
    file_id         VARCHAR(100),
    record_type     VARCHAR(10),        -- BPR, CLP, SVC, etc.
    record_content  VARCHAR(16000),
    line_number     INTEGER,
    ingested_at     TIMESTAMP_NTZ,
    source_path     VARCHAR(1000)
);

-- Alternative: JSON blobs for semi-structured data
CREATE TABLE LAKE_API.CUSTOMER_EVENTS_RAW (
    event_id        VARCHAR(100),
    event_type      VARCHAR(50),
    payload         VARIANT,            -- Snowflake JSON type
    received_at     TIMESTAMP_NTZ,
    api_endpoint    VARCHAR(500)
);
```

---

## STG Layer (Silver) - Complete Schema

Cleaned, validated, business-typed data. Deduplicated with audit columns.

```sql
-- Cleaned claim data with validation
CREATE TABLE STG_MEDEDI.CLAIM (
    -- Primary key: hash-based for idempotency
    claim_key           VARCHAR(64) PRIMARY KEY,        -- MD5(claim_id || file_id || line_num)

    -- Business identifiers
    claim_id            VARCHAR(50) NOT NULL,           -- Natural key from source
    patient_id          VARCHAR(50),                    -- FK to patient
    provider_npi        VARCHAR(10),                    -- National Provider Identifier
    payer_id            VARCHAR(50),                    -- FK to payer

    -- Claim details (typed and validated)
    service_date        DATE NOT NULL,                  -- Date of service
    submission_date     DATE,                           -- When claim submitted
    total_charge        NUMBER(12,2) NOT NULL,          -- Must be >= 0
    diagnosis_codes     ARRAY,                          -- Structured: ['J45.909', 'E11.9']
    procedure_codes     ARRAY,                          -- Structured: ['99213', '36415']

    -- Claim status
    status              VARCHAR(20),                    -- SUBMITTED, PAID, DENIED
    denial_reason       VARCHAR(500),                   -- If denied, reason code + text

    -- Audit columns (required on all STG tables)
    source_file_id      VARCHAR(100) NOT NULL,          -- Traceability to LAKE
    source_line_number  INTEGER,                        -- Line in LAKE file

    -- SCD Type 2 columns (track history)
    valid_from          TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
    valid_to            TIMESTAMP_NTZ DEFAULT '9999-12-31 23:59:59',
    is_current          BOOLEAN DEFAULT TRUE,

    -- Load metadata
    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
    loaded_by           VARCHAR(100) DEFAULT CURRENT_USER,
    etl_batch_id        VARCHAR(100)                    -- For reconciliation
);

-- Indexes for performance
CREATE INDEX idx_claim_patient ON STG_MEDEDI.CLAIM (patient_id);
CREATE INDEX idx_claim_provider ON STG_MEDEDI.CLAIM (provider_npi);
CREATE INDEX idx_claim_service_date ON STG_MEDEDI.CLAIM (service_date);

-- Payment data (from 835 remittance)
CREATE TABLE STG_MEDEDI.PAYMENT (
    payment_key         VARCHAR(64) PRIMARY KEY,
    payment_id          VARCHAR(50) NOT NULL,
    claim_id            VARCHAR(50) NOT NULL,           -- FK to CLAIM
    payer_id            VARCHAR(50),

    paid_amount         NUMBER(12,2) NOT NULL,
    payment_date        DATE NOT NULL,
    payment_method      VARCHAR(20),                    -- CHECK, EFT, CARD
    check_number        VARCHAR(50),

    -- Adjustment details
    adjustment_amount   NUMBER(12,2),
    adjustment_reason   VARCHAR(500),

    -- Audit columns
    source_file_id      VARCHAR(100) NOT NULL,
    valid_from          TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
    valid_to            TIMESTAMP_NTZ DEFAULT '9999-12-31 23:59:59',
    is_current          BOOLEAN DEFAULT TRUE,
    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Key STG characteristics:**

1. **Deduplicated** - Hash keys prevent duplicate rows
2. **Data types enforced** - DATE, NUMBER, BOOLEAN (not VARCHAR for everything)
3. **Validation rules applied** - Constraints, CHECK conditions
4. **SCD Type 2 ready** - valid_from/valid_to for history
5. **Audit trail** - Source lineage back to LAKE layer

---

## EDW Layer (Gold) - Complete Schema

Business-ready, aggregated, enriched data optimized for analytics.

```sql
-- Fact table: claim line items (grain: one row per line)
CREATE TABLE EDW_MEDEDI.FACT_CLAIM_LINE (
    -- Surrogate keys (FK to dimensions)
    claim_line_sk       INTEGER PRIMARY KEY,
    claim_sk            INTEGER NOT NULL,               -- FK to DIM_CLAIM
    date_sk             INTEGER NOT NULL,               -- FK to DIM_DATE (service date)
    procedure_sk        INTEGER NOT NULL,               -- FK to DIM_PROCEDURE
    provider_sk         INTEGER NOT NULL,               -- FK to DIM_PROVIDER
    patient_sk          INTEGER NOT NULL,               -- FK to DIM_PATIENT
    payer_sk            INTEGER NOT NULL,               -- FK to DIM_PAYER

    -- Degenerate dimensions (no separate table needed)
    claim_id            VARCHAR(50) NOT NULL,           -- Natural key
    line_number         INTEGER NOT NULL,

    -- Measures (additive facts - can be SUMmed)
    charge_amount       NUMBER(12,2) NOT NULL,
    paid_amount         NUMBER(12,2) NOT NULL,
    adjustment_amount   NUMBER(12,2),
    units               INTEGER,                        -- Service units

    -- Semi-additive measures (sum across some dimensions only)
    patient_responsibility NUMBER(12,2),                -- Don't sum across patients

    -- Non-additive measures (ratios, percentages - NEVER sum)
    payment_rate        NUMBER(5,4),                    -- paid_amount / charge_amount

    -- Pre-calculated dimensions (avoid JOINs in BI)
    denial_flag         BOOLEAN,
    days_to_payment     INTEGER,                        -- submission → payment

    -- Audit
    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP
);

-- Fact table: daily summary (grain: one row per provider + date)
CREATE TABLE EDW_MEDEDI.FACT_DAILY_SUMMARY (
    daily_summary_sk    INTEGER PRIMARY KEY,
    date_sk             INTEGER NOT NULL,
    provider_sk         INTEGER NOT NULL,

    -- Aggregated measures
    total_claims        INTEGER,
    total_charges       NUMBER(14,2),
    total_payments      NUMBER(14,2),
    total_adjustments   NUMBER(14,2),

    -- Calculated metrics
    avg_charge_per_claim NUMBER(12,2),
    payment_rate        NUMBER(5,4),
    denial_count        INTEGER,
    denial_rate         NUMBER(5,4),

    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (date_sk, provider_sk)                       -- Enforce grain
);

-- Dimension: Provider (SCD Type 2 for history)
CREATE TABLE EDW_MEDEDI.DIM_PROVIDER (
    provider_sk         INTEGER PRIMARY KEY,            -- Surrogate key
    provider_npi        VARCHAR(10) NOT NULL,           -- Natural key
    provider_name       VARCHAR(200),
    specialty           VARCHAR(100),
    subspecialty        VARCHAR(100),

    -- Address (current)
    address_line1       VARCHAR(200),
    address_line2       VARCHAR(200),
    address_city        VARCHAR(100),
    address_state       VARCHAR(2),
    address_zip         VARCHAR(10),

    -- Categorization (for grouping in reports)
    provider_type       VARCHAR(50),                    -- Individual, Group, Hospital
    taxonomy_code       VARCHAR(20),

    -- SCD Type 2 columns
    valid_from          DATE NOT NULL,
    valid_to            DATE NOT NULL DEFAULT '9999-12-31',
    is_current          BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit
    source_system       VARCHAR(50),
    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP
);

-- Index for SCD Type 2 lookups
CREATE INDEX idx_provider_npi_current ON EDW_MEDEDI.DIM_PROVIDER (provider_npi, is_current);

-- Dimension: Date (pre-populated, no SCD)
CREATE TABLE EDW_MEDEDI.DIM_DATE (
    date_sk             INTEGER PRIMARY KEY,            -- YYYYMMDD format (20260115)
    full_date           DATE UNIQUE NOT NULL,

    -- Calendar attributes
    year                INTEGER NOT NULL,
    quarter             INTEGER NOT NULL,               -- 1-4
    month               INTEGER NOT NULL,               -- 1-12
    month_name          VARCHAR(20),                    -- 'January'
    month_abbr          VARCHAR(3),                     -- 'Jan'
    week_of_year        INTEGER,                        -- 1-53
    day_of_month        INTEGER,                        -- 1-31
    day_of_week         INTEGER,                        -- 1=Monday
    day_name            VARCHAR(20),                    -- 'Monday'
    day_abbr            VARCHAR(3),                     -- 'Mon'

    -- Flags
    is_weekend          BOOLEAN NOT NULL,
    is_holiday          BOOLEAN NOT NULL,
    is_business_day     BOOLEAN NOT NULL,

    -- Fiscal calendar (if different from calendar)
    fiscal_year         INTEGER,
    fiscal_quarter      INTEGER,
    fiscal_month        INTEGER,

    -- Relative dates (useful for filters)
    is_current_day      BOOLEAN,
    is_current_month    BOOLEAN,
    is_current_quarter  BOOLEAN,
    is_current_year     BOOLEAN
);

-- Dimension: Patient (SCD Type 2 for demographics changes)
CREATE TABLE EDW_MEDEDI.DIM_PATIENT (
    patient_sk          INTEGER PRIMARY KEY,
    patient_id          VARCHAR(50) NOT NULL,           -- Natural key (MRN, etc.)

    -- Demographics
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    date_of_birth       DATE,
    gender              VARCHAR(10),

    -- Contact
    email               VARCHAR(255),
    phone               VARCHAR(20),
    address_city        VARCHAR(100),
    address_state       VARCHAR(2),
    address_zip         VARCHAR(10),

    -- Insurance
    primary_insurance   VARCHAR(100),
    secondary_insurance VARCHAR(100),

    -- SCD Type 2
    valid_from          DATE NOT NULL,
    valid_to            DATE NOT NULL DEFAULT '9999-12-31',
    is_current          BOOLEAN NOT NULL DEFAULT TRUE,

    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP
);

-- Dimension: Payer (relatively static, SCD Type 1)
CREATE TABLE EDW_MEDEDI.DIM_PAYER (
    payer_sk            INTEGER PRIMARY KEY,
    payer_id            VARCHAR(50) UNIQUE NOT NULL,    -- Natural key
    payer_name          VARCHAR(200) NOT NULL,
    payer_type          VARCHAR(50),                    -- Commercial, Medicare, Medicaid

    -- Contact
    payer_address       VARCHAR(500),
    payer_phone         VARCHAR(20),
    payer_website       VARCHAR(255),

    -- Categorization
    network_status      VARCHAR(20),                    -- In-Network, Out-of-Network
    priority_level      INTEGER,                        -- For COB

    -- Audit
    effective_date      DATE,
    termination_date    DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP
);

-- Dimension: Procedure (SCD Type 1, codes change infrequently)
CREATE TABLE EDW_MEDEDI.DIM_PROCEDURE (
    procedure_sk        INTEGER PRIMARY KEY,
    procedure_code      VARCHAR(20) UNIQUE NOT NULL,    -- CPT, HCPCS
    procedure_desc      VARCHAR(500),
    procedure_type      VARCHAR(50),                    -- CPT, HCPCS Level I/II

    -- Categorization
    category            VARCHAR(100),
    subcategory         VARCHAR(100),

    -- Pricing reference
    standard_charge     NUMBER(10,2),                   -- Typical charge
    rvu                 NUMBER(8,2),                    -- Relative Value Unit

    -- Validity
    effective_date      DATE,
    termination_date    DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP
);
```

**Key EDW characteristics:**

1. **Star schema** - Facts surrounded by dimensions
2. **Surrogate keys** - Integer keys for dimensions (not natural keys)
3. **Pre-joined/aggregated** - Optimized for BI query performance
4. **Conformed dimensions** - Shared dimensions (DIM_DATE, DIM_PROVIDER) across fact tables
5. **Grain enforcement** - UNIQUE constraints enforce one row per grain
6. **Pre-calculated metrics** - Ratios, flags computed once at load time

---

## Loading Patterns

### LAKE → STG

```sql
-- Idempotent load with hash-based deduplication
INSERT INTO STG_MEDEDI.CLAIM
SELECT
    MD5(claim_id || '||' || file_id || '||' || line_number) AS claim_key,
    claim_id,
    patient_id,
    provider_npi,
    -- ... other columns
    file_id AS source_file_id,
    line_number AS source_line_number,
    CURRENT_TIMESTAMP AS load_timestamp
FROM LAKE_MEDEDI.EDI_837I_RAW
WHERE NOT EXISTS (
    SELECT 1 FROM STG_MEDEDI.CLAIM stg
    WHERE stg.claim_key = MD5(claim_id || '||' || file_id || '||' || line_number)
);
```

### STG → EDW

```sql
-- Populate fact table from staging
INSERT INTO EDW_MEDEDI.FACT_CLAIM_LINE
SELECT
    seq_claim_line_sk.NEXTVAL,                  -- Generate surrogate key
    dim_claim.claim_sk,                         -- Lookup dimension keys
    dim_date.date_sk,
    dim_procedure.procedure_sk,
    dim_provider.provider_sk,
    dim_patient.patient_sk,
    dim_payer.payer_sk,
    stg.claim_id,
    stg.line_number,
    stg.charge_amount,
    stg.paid_amount,
    stg.adjustment_amount,
    stg.units,
    stg.charge_amount - stg.paid_amount AS patient_responsibility,
    CASE WHEN stg.charge_amount > 0
         THEN stg.paid_amount / stg.charge_amount
         ELSE 0 END AS payment_rate,
    stg.denial_flag,
    DATEDIFF(day, stg.submission_date, stg.payment_date) AS days_to_payment,
    CURRENT_TIMESTAMP
FROM STG_MEDEDI.CLAIM_LINE stg
JOIN EDW_MEDEDI.DIM_DATE dim_date
    ON stg.service_date = dim_date.full_date
JOIN EDW_MEDEDI.DIM_PROVIDER dim_provider
    ON stg.provider_npi = dim_provider.provider_npi
    AND dim_provider.is_current = TRUE
-- ... other dimension JOINs
WHERE stg.is_current = TRUE
AND NOT EXISTS (
    SELECT 1 FROM EDW_MEDEDI.FACT_CLAIM_LINE fact
    WHERE fact.claim_id = stg.claim_id
    AND fact.line_number = stg.line_number
);
```

---

## Performance Optimization

### Clustering Keys

```sql
-- Snowflake: cluster by commonly filtered columns
ALTER TABLE EDW_MEDEDI.FACT_CLAIM_LINE
CLUSTER BY (date_sk, provider_sk);

ALTER TABLE STG_MEDEDI.CLAIM
CLUSTER BY (service_date, provider_npi);
```

### Materialized Views (for expensive aggregations)

```sql
CREATE MATERIALIZED VIEW EDW_MEDEDI.MV_PROVIDER_MONTHLY_SUMMARY AS
SELECT
    DATE_TRUNC('month', dim_date.full_date) AS month,
    dim_provider.provider_sk,
    dim_provider.provider_name,
    COUNT(DISTINCT fact.claim_line_sk) AS total_lines,
    SUM(fact.charge_amount) AS total_charges,
    SUM(fact.paid_amount) AS total_payments,
    AVG(fact.payment_rate) AS avg_payment_rate
FROM EDW_MEDEDI.FACT_CLAIM_LINE fact
JOIN EDW_MEDEDI.DIM_DATE dim_date ON fact.date_sk = dim_date.date_sk
JOIN EDW_MEDEDI.DIM_PROVIDER dim_provider ON fact.provider_sk = dim_provider.provider_sk
WHERE dim_provider.is_current = TRUE
GROUP BY 1, 2, 3;
```

---

## Summary

- **LAKE**: Raw, immutable, append-only, minimal schema
- **STG**: Cleaned, typed, validated, deduplicated, audit columns
- **EDW**: Star schema, surrogate keys, pre-aggregated, BI-ready
