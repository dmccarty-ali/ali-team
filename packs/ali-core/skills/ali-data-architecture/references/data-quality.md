# ali-data-architecture - Data Quality Reference

Complete validation rules and reconciliation patterns.

---

## Validation Framework

### Python Validation Implementation

```python
from dataclasses import dataclass
from typing import Optional, List
import re
from datetime import datetime, date

@dataclass
class ValidationResult:
    """Result of validation with errors."""
    is_valid: bool
    errors: List[str]
    warnings: List[str] = None

    def __post_init__(self):
        if self.warnings is None:
            self.warnings = []


def validate_claim(claim: dict) -> ValidationResult:
    """Validate healthcare claim record.

    Validation categories:
    1. Required fields
    2. Format validation (regex, patterns)
    3. Range validation (numeric bounds, date ranges)
    4. Referential integrity (FK existence)
    5. Business rules (cross-field logic)
    6. Data quality flags (warnings, not blockers)
    """
    errors = []
    warnings = []

    # 1. Required Fields
    required_fields = ['claim_id', 'patient_id', 'provider_npi', 'service_date', 'charge_amount']
    for field in required_fields:
        if not claim.get(field):
            errors.append(f"Required field missing: {field}")

    # 2. Format Validation
    # NPI must be 10 digits
    if claim.get('provider_npi'):
        if not re.match(r'^\d{10}$', claim['provider_npi']):
            errors.append(f"Invalid NPI format: {claim['provider_npi']} (must be 10 digits)")

    # Claim ID format (example: CLM followed by 10 digits)
    if claim.get('claim_id'):
        if not re.match(r'^CLM\d{10}$', claim['claim_id']):
            errors.append(f"Invalid claim_id format: {claim['claim_id']} (expected CLM + 10 digits)")

    # Phone format (if present)
    if claim.get('phone'):
        phone = re.sub(r'[^0-9]', '', claim['phone'])  # Remove non-digits
        if len(phone) != 10:
            errors.append(f"Invalid phone: {claim['phone']} (must be 10 digits)")

    # Email format (if present)
    if claim.get('email'):
        if not re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', claim['email']):
            errors.append(f"Invalid email format: {claim['email']}")

    # 3. Range Validation
    # Charge amount must be non-negative
    if claim.get('charge_amount') is not None:
        try:
            charge = float(claim['charge_amount'])
            if charge < 0:
                errors.append(f"charge_amount cannot be negative: {charge}")
            if charge > 1000000:  # Sanity check
                warnings.append(f"Unusually high charge_amount: {charge}")
        except ValueError:
            errors.append(f"charge_amount must be numeric: {claim['charge_amount']}")

    # Paid amount cannot exceed charge amount
    if claim.get('paid_amount') and claim.get('charge_amount'):
        try:
            paid = float(claim['paid_amount'])
            charge = float(claim['charge_amount'])
            if paid > charge:
                warnings.append(f"paid_amount ({paid}) exceeds charge_amount ({charge})")
        except ValueError:
            pass  # Already flagged above

    # Date range checks
    if claim.get('service_date'):
        try:
            service_date = datetime.strptime(str(claim['service_date']), '%Y-%m-%d').date()
            today = date.today()

            # Service date cannot be future
            if service_date > today:
                errors.append(f"service_date cannot be in future: {service_date}")

            # Service date cannot be too old (e.g., > 5 years)
            years_ago = (today - service_date).days / 365
            if years_ago > 5:
                warnings.append(f"service_date is {years_ago:.1f} years old: {service_date}")

        except ValueError:
            errors.append(f"Invalid service_date format: {claim['service_date']} (expected YYYY-MM-DD)")

    # 4. Referential Integrity (would query database in real implementation)
    # Check if diagnosis code is valid ICD-10
    if claim.get('diagnosis_code'):
        if not is_valid_icd10(claim['diagnosis_code']):
            errors.append(f"Invalid ICD-10 code: {claim['diagnosis_code']}")

    # Check if procedure code is valid CPT
    if claim.get('procedure_code'):
        if not is_valid_cpt(claim['procedure_code']):
            errors.append(f"Invalid CPT code: {claim['procedure_code']}")

    # 5. Business Rules
    # service_date must be before submission_date
    if claim.get('service_date') and claim.get('submission_date'):
        try:
            service_date = datetime.strptime(str(claim['service_date']), '%Y-%m-%d').date()
            submission_date = datetime.strptime(str(claim['submission_date']), '%Y-%m-%d').date()
            if service_date > submission_date:
                errors.append(f"service_date ({service_date}) cannot be after submission_date ({submission_date})")
        except ValueError:
            pass  # Date format errors already caught

    # submission_date must be before payment_date (if both present)
    if claim.get('submission_date') and claim.get('payment_date'):
        try:
            submission_date = datetime.strptime(str(claim['submission_date']), '%Y-%m-%d').date()
            payment_date = datetime.strptime(str(claim['payment_date']), '%Y-%m-%D').date()
            if submission_date > payment_date:
                errors.append(f"submission_date ({submission_date}) cannot be after payment_date ({payment_date})")
        except ValueError:
            pass

    # If status is DENIED, denial_reason must be present
    if claim.get('status') == 'DENIED' and not claim.get('denial_reason'):
        errors.append("denial_reason required when status is DENIED")

    # 6. Data Quality Flags (warnings)
    # Missing optional but recommended fields
    if not claim.get('patient_ssn'):
        warnings.append("patient_ssn missing (recommended for matching)")

    if not claim.get('diagnosis_code'):
        warnings.append("diagnosis_code missing (recommended for analytics)")

    # Check for placeholder values
    if claim.get('patient_id') in ['0', '00000', 'UNKNOWN', 'N/A']:
        warnings.append(f"Placeholder patient_id detected: {claim['patient_id']}")

    return ValidationResult(
        is_valid=len(errors) == 0,
        errors=errors,
        warnings=warnings
    )


def is_valid_icd10(code: str) -> bool:
    """Validate ICD-10 code format and existence."""
    # Basic format check (would query reference table in real implementation)
    return bool(re.match(r'^[A-Z][0-9]{2}(\.[0-9]{1,4})?$', code))


def is_valid_cpt(code: str) -> bool:
    """Validate CPT code format and existence."""
    # Basic format check (5 digits, would query reference table in real implementation)
    return bool(re.match(r'^\d{5}$', code))


# Example usage
claim = {
    'claim_id': 'CLM1234567890',
    'patient_id': 'P001',
    'provider_npi': '1234567890',
    'service_date': '2026-01-15',
    'charge_amount': 150.00,
    'diagnosis_code': 'J45.909',  # Asthma
    'procedure_code': '99213'     # Office visit
}

result = validate_claim(claim)
if result.is_valid:
    print("✓ Claim valid")
else:
    print("✗ Claim validation failed:")
    for error in result.errors:
        print(f"  - {error}")

if result.warnings:
    print("⚠ Warnings:")
    for warning in result.warnings:
        print(f"  - {warning}")
```

---

## SQL-Based Validation

### Constraint-Based Validation

```sql
-- Add constraints to enforce validation at database level
ALTER TABLE STG_MEDEDI.CLAIM ADD CONSTRAINT chk_charge_nonnegative
    CHECK (charge_amount >= 0);

ALTER TABLE STG_MEDEDI.CLAIM ADD CONSTRAINT chk_service_before_submission
    CHECK (service_date <= submission_date);

ALTER TABLE STG_MEDEDI.CLAIM ADD CONSTRAINT chk_npi_format
    CHECK (LENGTH(provider_npi) = 10 AND provider_npi ~ '^[0-9]+$');

-- Referential integrity
ALTER TABLE STG_MEDEDI.CLAIM ADD CONSTRAINT fk_provider
    FOREIGN KEY (provider_npi) REFERENCES DIM_PROVIDER (provider_npi);
```

### Validation Query Pattern

```sql
-- Identify invalid records in staging table
CREATE TABLE validation_errors AS
SELECT
    claim_id,
    'charge_negative' AS error_type,
    'Charge amount is negative' AS error_message
FROM STG_MEDEDI.CLAIM
WHERE charge_amount < 0

UNION ALL

SELECT
    claim_id,
    'service_after_submission',
    'Service date is after submission date'
FROM STG_MEDEDI.CLAIM
WHERE service_date > submission_date

UNION ALL

SELECT
    claim_id,
    'invalid_npi',
    'NPI format invalid'
FROM STG_MEDEDI.CLAIM
WHERE LENGTH(provider_npi) != 10
   OR provider_npi NOT REGEXP '^[0-9]{10}$'

UNION ALL

SELECT
    claim_id,
    'missing_required_field',
    'Required field missing: ' || field_name
FROM (
    SELECT claim_id, 'patient_id' AS field_name FROM STG_MEDEDI.CLAIM WHERE patient_id IS NULL
    UNION ALL
    SELECT claim_id, 'provider_npi' FROM STG_MEDEDI.CLAIM WHERE provider_npi IS NULL
    UNION ALL
    SELECT claim_id, 'service_date' FROM STG_MEDEDI.CLAIM WHERE service_date IS NULL
);

-- Reject records with errors
DELETE FROM STG_MEDEDI.CLAIM
WHERE claim_id IN (SELECT claim_id FROM validation_errors);

-- Log errors
INSERT INTO SYS_CONTROL.VALIDATION_LOG (
    table_name,
    record_id,
    error_type,
    error_message,
    validation_timestamp
)
SELECT
    'STG_MEDEDI.CLAIM',
    claim_id,
    error_type,
    error_message,
    CURRENT_TIMESTAMP
FROM validation_errors;
```

---

## Reconciliation

### Row Count Reconciliation

```sql
-- Compare row counts between layers
SELECT
    'LAKE' AS layer,
    COUNT(*) AS row_count,
    COUNT(DISTINCT claim_id) AS unique_claims,
    MIN(ingested_at) AS earliest_record,
    MAX(ingested_at) AS latest_record
FROM LAKE_MEDEDI.EDI_837I_RAW
WHERE ingested_at::DATE = CURRENT_DATE

UNION ALL

SELECT
    'STG' AS layer,
    COUNT(*) AS row_count,
    COUNT(DISTINCT claim_id) AS unique_claims,
    MIN(load_timestamp) AS earliest_record,
    MAX(load_timestamp) AS latest_record
FROM STG_MEDEDI.CLAIM
WHERE load_timestamp::DATE = CURRENT_DATE

UNION ALL

SELECT
    'EDW' AS layer,
    COUNT(*) AS row_count,
    COUNT(DISTINCT claim_id) AS unique_claims,
    MIN(load_timestamp) AS earliest_record,
    MAX(load_timestamp) AS latest_record
FROM EDW_MEDEDI.FACT_CLAIM
WHERE load_timestamp::DATE = CURRENT_DATE;

-- Alert if counts don't match (expect some filtering in STG/EDW)
-- Rule: STG count <= LAKE count (some records filtered out)
--       EDW count <= STG count (some aggregation may occur)
```

### Amount Reconciliation

```sql
-- Financial reconciliation: total charges
SELECT
    'LAKE' AS layer,
    SUM(CAST(charge_amount AS NUMBER(12,2))) AS total_charges
FROM LAKE_MEDEDI.CLAIM_RAW
WHERE load_date = CURRENT_DATE

UNION ALL

SELECT
    'STG' AS layer,
    SUM(charge_amount) AS total_charges
FROM STG_MEDEDI.CLAIM
WHERE load_timestamp::DATE = CURRENT_DATE

UNION ALL

SELECT
    'EDW' AS layer,
    SUM(charge_amount) AS total_charges
FROM EDW_MEDEDI.FACT_CLAIM
WHERE load_timestamp::DATE = CURRENT_DATE;

-- Identify discrepancies
WITH layer_totals AS (
    SELECT layer, total_charges FROM -- above query
)
SELECT
    a.layer AS layer_a,
    b.layer AS layer_b,
    a.total_charges AS total_a,
    b.total_charges AS total_b,
    a.total_charges - b.total_charges AS difference,
    CASE
        WHEN b.total_charges = 0 THEN NULL
        ELSE ((a.total_charges - b.total_charges) / b.total_charges) * 100
    END AS pct_difference
FROM layer_totals a
CROSS JOIN layer_totals b
WHERE a.layer != b.layer;
```

### Claim-Level Reconciliation

```sql
-- Identify claims in LAKE but not in STG
SELECT
    lake.claim_id,
    lake.file_id,
    'Missing in STG' AS reconciliation_issue
FROM LAKE_MEDEDI.CLAIM_RAW lake
LEFT JOIN STG_MEDEDI.CLAIM stg
    ON lake.claim_id = stg.claim_id
WHERE lake.load_date = CURRENT_DATE
  AND stg.claim_id IS NULL

UNION ALL

-- Identify claims in STG but not in EDW
SELECT
    stg.claim_id,
    stg.source_file_id,
    'Missing in EDW' AS reconciliation_issue
FROM STG_MEDEDI.CLAIM stg
LEFT JOIN EDW_MEDEDI.FACT_CLAIM edw
    ON stg.claim_id = edw.claim_id
WHERE stg.load_timestamp::DATE = CURRENT_DATE
  AND edw.claim_id IS NULL;
```

### Reconciliation Checkpoints

```sql
-- Define reconciliation checkpoints
CREATE TABLE SYS_CONTROL.RECONCILIATION_CHECKPOINTS (
    checkpoint_id       VARCHAR(100) PRIMARY KEY,
    table_name          VARCHAR(200) NOT NULL,
    metric_type         VARCHAR(50) NOT NULL,  -- ROW_COUNT, SUM, AVG, etc.
    metric_column       VARCHAR(100),          -- Column to measure (for SUM, AVG)
    expected_value      NUMBER(18,2),
    actual_value        NUMBER(18,2),
    tolerance_pct       NUMBER(5,2),           -- Allowed variance (e.g., 1.0 = 1%)
    status              VARCHAR(20),           -- PASS, FAIL, WARNING
    checkpoint_timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
    load_batch_id       VARCHAR(100)
);

-- Insert checkpoint
INSERT INTO SYS_CONTROL.RECONCILIATION_CHECKPOINTS (
    checkpoint_id,
    table_name,
    metric_type,
    metric_column,
    expected_value,
    actual_value,
    tolerance_pct,
    status,
    load_batch_id
)
SELECT
    'STG_CLAIM_' || CURRENT_DATE || '_ROW_COUNT',
    'STG_MEDEDI.CLAIM',
    'ROW_COUNT',
    NULL,
    (SELECT COUNT(*) FROM LAKE_MEDEDI.CLAIM_RAW WHERE load_date = CURRENT_DATE) AS expected,
    (SELECT COUNT(*) FROM STG_MEDEDI.CLAIM WHERE load_timestamp::DATE = CURRENT_DATE) AS actual,
    5.0,  -- Allow 5% variance
    CASE
        WHEN ABS(actual - expected) / expected * 100 <= 5.0 THEN 'PASS'
        WHEN ABS(actual - expected) / expected * 100 <= 10.0 THEN 'WARNING'
        ELSE 'FAIL'
    END,
    CURRENT_BATCH_ID
FROM (SELECT 1);  -- Dummy for inline calculation

-- Query failed checkpoints
SELECT *
FROM SYS_CONTROL.RECONCILIATION_CHECKPOINTS
WHERE status = 'FAIL'
  AND checkpoint_timestamp >= CURRENT_DATE;
```

---

## Data Profiling

### Profile Statistics

```sql
-- Generate data profile for a table
CREATE OR REPLACE PROCEDURE profile_table(table_name VARCHAR)
RETURNS TABLE (
    column_name VARCHAR,
    data_type VARCHAR,
    null_count INTEGER,
    null_pct NUMBER(5,2),
    distinct_count INTEGER,
    distinct_pct NUMBER(5,2),
    min_value VARCHAR,
    max_value VARCHAR,
    avg_value NUMBER(18,2),
    std_dev NUMBER(18,2)
)
AS
$$
BEGIN
    RETURN TABLE(
        SELECT
            column_name,
            data_type,
            COUNT(*) - COUNT(column_name) AS null_count,
            ((COUNT(*) - COUNT(column_name)) / COUNT(*)) * 100 AS null_pct,
            COUNT(DISTINCT column_name) AS distinct_count,
            (COUNT(DISTINCT column_name) / COUNT(*)) * 100 AS distinct_pct,
            MIN(column_name)::VARCHAR AS min_value,
            MAX(column_name)::VARCHAR AS max_value,
            AVG(CASE WHEN IS_NUMBER(column_name) THEN column_name::NUMBER END) AS avg_value,
            STDDEV(CASE WHEN IS_NUMBER(column_name) THEN column_name::NUMBER END) AS std_dev
        FROM table_name
        GROUP BY column_name, data_type
    );
END;
$$;

-- Run profile
CALL profile_table('STG_MEDEDI.CLAIM');
```

---

## Anomaly Detection

### Statistical Outliers

```sql
-- Detect outliers using Z-score
WITH stats AS (
    SELECT
        AVG(charge_amount) AS mean,
        STDDEV(charge_amount) AS stddev
    FROM STG_MEDEDI.CLAIM
)
SELECT
    claim_id,
    charge_amount,
    (charge_amount - stats.mean) / stats.stddev AS z_score
FROM STG_MEDEDI.CLAIM, stats
WHERE ABS((charge_amount - stats.mean) / stats.stddev) > 3  -- > 3 std devs = outlier
ORDER BY z_score DESC;
```

### Duplicate Detection

```sql
-- Identify duplicate claims
SELECT
    claim_id,
    patient_id,
    service_date,
    COUNT(*) AS duplicate_count
FROM STG_MEDEDI.CLAIM
GROUP BY claim_id, patient_id, service_date
HAVING COUNT(*) > 1;
```

---

## Summary

- **Validation**: Required fields, format, range, referential integrity, business rules
- **Reconciliation**: Row counts, amounts, claim-level across layers
- **Profiling**: Column statistics, nulls, distinct values, distributions
- **Anomaly detection**: Outliers, duplicates, unexpected patterns
