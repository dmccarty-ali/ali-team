# ali-sap-mdg - Data Quality Reference

This reference contains detailed Data Quality Management (DQM) configuration, validation rules, and address standardization examples.

---

## Data Quality Management (DQM) Overview

DQM is SAP's built-in data quality tooling for validation, standardization, and enrichment of master data.

**Key Capabilities:**
- Address validation and standardization
- Duplicate detection
- Data profiling and cleansing
- Integration with external data providers
- Batch and real-time validation

---

## DQM Integration Points

| Integration Point | Purpose | Example |
|-------------------|---------|---------|
| **Create Validation** | Block invalid data at creation | Reject customer if address undeliverable |
| **Update Validation** | Validate changes before approval | Warn if new address not DQM-validated |
| **Batch Cleansing** | Periodically cleanse existing data | Monthly job to standardize all addresses |
| **Duplicate Detection** | Find potential duplicates | Nightly job to flag possible duplicate BPs |

---

## Validation Rules Code Examples

### Example 1: Tax ID Format Validation

```abap
* Example validation rule in USMD_RULE
VALIDATION: CUSTOMER_TAX_ID_FORMAT
  IF customer.COUNTRY = 'US'
  THEN
    ASSERT REGEX_MATCH(customer.TAX_ID, '^\d{2}-\d{7}$')
    ERROR_MESSAGE = 'US Tax ID must be format: 12-3456789'
  ELSE IF customer.COUNTRY = 'DE'
  THEN
    ASSERT REGEX_MATCH(customer.TAX_ID, '^\d{11}$')
    ERROR_MESSAGE = 'German Tax ID must be 11 digits'
  ELSE IF customer.COUNTRY = 'GB'
  THEN
    ASSERT REGEX_MATCH(customer.TAX_ID, '^[A-Z]{2}\d{9}$')
    ERROR_MESSAGE = 'UK Tax ID must be format: AB123456789'
```

### Example 2: Required Field Validation

```abap
VALIDATION: CUSTOMER_REQUIRED_FIELDS
  " Name is always required
  IF customer.NAME IS EMPTY
    ERROR_MESSAGE = 'Customer name is required'
    SEVERITY = 'ERROR'

  " Tax ID required for credit customers
  IF customer.CUSTOMER_TYPE = 'CREDIT'
    AND customer.TAX_ID IS EMPTY
  THEN
    ERROR_MESSAGE = 'Tax ID is required for credit customers'
    SEVERITY = 'ERROR'

  " Email required for B2B customers
  IF customer.SEGMENT = 'B2B'
    AND customer.EMAIL IS EMPTY
  THEN
    ERROR_MESSAGE = 'Email is required for B2B customers'
    SEVERITY = 'WARNING'  " Warning, not blocking
```

### Example 3: Cross-Field Validation

```abap
VALIDATION: CUSTOMER_CREDIT_LIMIT_VALIDATION
  " Credit limit must be consistent with customer risk rating
  IF customer.RISK_RATING = 'HIGH'
    AND customer.CREDIT_LIMIT > 500000
  THEN
    ERROR_MESSAGE = 'High-risk customers cannot have credit limit >$500K'
    SEVERITY = 'ERROR'

  " Payment terms and credit limit must align
  IF customer.PAYMENT_TERMS_DAYS > 90
    AND customer.CREDIT_LIMIT > 1000000
  THEN
    ERROR_MESSAGE = 'Extended payment terms (>90 days) not allowed for credit limit >$1M'
    SEVERITY = 'ERROR'
```

### Example 4: External API Validation

```abap
VALIDATION: CUSTOMER_TAX_ID_VERIFICATION
  " Call external tax authority API to verify Tax ID
  IF customer.COUNTRY = 'US'
  THEN
    response = HTTP_CALL(
      url = 'https://irs.gov/api/verify-tin',
      method = 'POST',
      body = { 'tin': customer.TAX_ID, 'name': customer.NAME }
    )

    IF response.status != 'VERIFIED'
    THEN
      ERROR_MESSAGE = 'Tax ID could not be verified with IRS: ' + response.reason
      SEVERITY = 'ERROR'
```

---

## Address Standardization Examples

### Before DQM (Unstandardized Input)

**Raw customer input:**
```
Customer Name: acme corp
Street: 123 main st apt 5
City: anytown
State: ca
Zip: 90210
```

### After DQM Validation

**Standardized output:**
```
Customer Name: Acme Corporation
Address:
  Street: 123 Main Street
  Unit: Apartment 5
  City: Anytown
  State: CA
  Postal Code: 90210-1234
  Country: US
Geocode:
  Latitude: 34.0522
  Longitude: -118.2437
Validation:
  Deliverable: TRUE
  Quality Score: 98%
  Provider: Loqate
  Validated At: 2026-02-16T10:30:00Z
```

### Address Validation Scenarios

#### Scenario 1: Valid Address (Auto-Approve)

**Input:**
```
Street: 1600 Pennsylvania Avenue NW
City: Washington
State: DC
Zip: 20500
```

**DQM Result:**
```json
{
  "match_quality": 1.0,
  "deliverable": true,
  "standardized_address": {
    "street": "1600 Pennsylvania Avenue Northwest",
    "city": "Washington",
    "state": "DC",
    "postal_code": "20500-0005",
    "country": "US"
  },
  "geocode": {
    "latitude": 38.8977,
    "longitude": -77.0365
  },
  "action": "AUTO_APPROVE"
}
```

#### Scenario 2: Undeliverable Address (Block)

**Input:**
```
Street: 123 Fake Street
City: Nowhere
State: XX
Zip: 00000
```

**DQM Result:**
```json
{
  "match_quality": 0.0,
  "deliverable": false,
  "error": "Address not found in postal database",
  "action": "BLOCK",
  "message": "Address is undeliverable. Please verify and correct."
}
```

#### Scenario 3: Partial Match (Manual Review)

**Input:**
```
Street: 123 Main St
City: Springfield
State: IL
Zip: 62701
```

**DQM Result:**
```json
{
  "match_quality": 0.85,
  "deliverable": true,
  "candidates": [
    {
      "street": "123 East Main Street",
      "city": "Springfield",
      "state": "IL",
      "postal_code": "62701-1234"
    },
    {
      "street": "123 West Main Street",
      "city": "Springfield",
      "state": "IL",
      "postal_code": "62701-5678"
    }
  ],
  "action": "MANUAL_REVIEW",
  "message": "Multiple addresses match. Steward review required."
}
```

---

## DQM Provider Configuration

### Supported Providers

| Provider | Coverage | Strengths | Cost |
|----------|----------|-----------|------|
| **Loqate** | Global (240+ countries) | Excellent international coverage, real-time | Medium |
| **Melissa Data** | US, Canada, UK | High accuracy in North America | Low-Medium |
| **Informatica** | Global | Enterprise-grade, advanced cleansing | High |
| **Postcode Anywhere** | UK, Europe | Best for UK/EU addresses | Low |

### Configuration Example (Loqate)

**Transaction:** DQM_MAIN

```yaml
Provider Configuration:
  Name: Loqate
  Type: Cloud API
  Endpoint: https://api.addressy.com/Capture/Interactive/Find/v1.1/json
  Authentication:
    Type: API Key
    Key: <stored in SAP Secure Storage>

  Supported Countries:
    - US
    - CA
    - GB
    - FR
    - DE
    - AU

  Features Enabled:
    - Address validation: YES
    - Address standardization: YES
    - Geocoding: YES
    - Type-ahead search: NO (not used in MDG)

  Performance:
    Timeout: 5 seconds
    Retry: 3 attempts (exponential backoff)
    Cache: 24 hours (for repeated lookups)
```

---

## Batch Data Cleansing

### Monthly Address Cleansing Job

**Purpose:** Periodically cleanse existing customer addresses to maintain data quality

**Implementation:**

```abap
* Report: Z_MDG_ADDRESS_CLEANSING
* Schedule: Monthly (1st day of month, 2 AM)

REPORT z_mdg_address_cleansing.

DATA: lt_customers TYPE TABLE OF customer,
      ls_customer TYPE customer,
      lv_dqm_result TYPE dqm_result,
      lv_updated_count TYPE i,
      lv_error_count TYPE i.

" Select all customers with addresses modified >30 days ago
SELECT * FROM customer
  INTO TABLE lt_customers
  WHERE last_address_update < sy-datum - 30.

LOOP AT lt_customers INTO ls_customer.

  " Call DQM validation
  CALL FUNCTION 'DQM_VALIDATE_ADDRESS'
    EXPORTING
      address = ls_customer-address
    IMPORTING
      result = lv_dqm_result.

  " Update if quality score improved
  IF lv_dqm_result-quality_score > ls_customer-address_quality_score
    AND lv_dqm_result-deliverable = abap_true.

    " Update customer with standardized address
    UPDATE customer
      SET street = lv_dqm_result-standardized_street
          city = lv_dqm_result-standardized_city
          postal_code = lv_dqm_result-standardized_postal
          latitude = lv_dqm_result-latitude
          longitude = lv_dqm_result-longitude
          address_quality_score = lv_dqm_result-quality_score
          last_address_update = sy-datum
      WHERE customer_id = ls_customer-customer_id.

    ADD 1 TO lv_updated_count.

  ELSE IF lv_dqm_result-deliverable = abap_false.
    " Flag undeliverable addresses for manual review
    INSERT INTO dqm_review_queue
      VALUES ( customer_id = ls_customer-customer_id
               reason = 'Undeliverable address'
               priority = 'HIGH' ).

    ADD 1 TO lv_error_count.
  ENDIF.

ENDLOOP.

" Log results
WRITE: / 'Address cleansing complete:',
       / 'Updated:', lv_updated_count,
       / 'Flagged for review:', lv_error_count.
```

---

## Duplicate Detection Configuration

### Duplicate Detection Job

**Purpose:** Nightly job to identify potential duplicate customers

```abap
* Report: Z_MDG_DUPLICATE_DETECTION
* Schedule: Nightly (2 AM)

PARAMETERS: p_score TYPE i DEFAULT 70.  " Minimum match score

" Use MDG matching engine to find duplicates
SELECT * FROM customer
  WHERE created_date >= sy-datum - 1  " Only check new customers
  INTO TABLE @DATA(lt_new_customers).

LOOP AT lt_new_customers INTO DATA(ls_new).

  " Compare against existing customers
  SELECT * FROM customer
    WHERE customer_id != ls_new-customer_id
    INTO TABLE @DATA(lt_existing).

  LOOP AT lt_existing INTO DATA(ls_existing).

    " Calculate match score
    DATA(lv_score) = calculate_match_score(
      source = ls_new
      target = ls_existing
    ).

    " If score exceeds threshold, add to manual review queue
    IF lv_score >= p_score.
      INSERT INTO mdg_match_review_queue
        VALUES (
          candidate1 = ls_new-customer_id
          candidate2 = ls_existing-customer_id
          match_score = lv_score
          status = 'PENDING'
          created_date = sy-datum
          created_time = sy-uzeit
        ).
    ENDIF.

  ENDLOOP.
ENDLOOP.
```

---

## Data Quality Metrics

### Key Metrics to Track

| Metric | Target | Frequency |
|--------|--------|-----------|
| **Address Validation Rate** | >98% | Daily |
| **Deliverable Address Rate** | >95% | Daily |
| **Duplicate Detection Rate** | <2% | Weekly |
| **Data Completeness** | >90% | Weekly |
| **Data Accuracy** | >95% | Monthly |

### Quality Dashboard

```sql
-- Example SQL for quality metrics dashboard
SELECT
  COUNT(*) AS total_customers,
  SUM(CASE WHEN address_validated = 'X' THEN 1 ELSE 0 END) AS validated_addresses,
  SUM(CASE WHEN address_deliverable = 'X' THEN 1 ELSE 0 END) AS deliverable_addresses,
  AVG(address_quality_score) AS avg_quality_score,
  SUM(CASE WHEN tax_id IS NOT NULL THEN 1 ELSE 0 END) AS with_tax_id,
  SUM(CASE WHEN email IS NOT NULL THEN 1 ELSE 0 END) AS with_email
FROM customer
WHERE active = 'X';

-- Calculate validation rate
SELECT
  (validated_addresses::float / total_customers * 100) AS validation_rate_pct,
  (deliverable_addresses::float / total_customers * 100) AS deliverable_rate_pct,
  (with_tax_id::float / total_customers * 100) AS completeness_tax_id_pct,
  (with_email::float / total_customers * 100) AS completeness_email_pct
FROM customer_quality_summary;
```

---

## Best Practices

### 1. Validation Timing

| Validation Type | When | Why |
|-----------------|------|-----|
| **Blocking (Error)** | Create, Update | Prevent bad data from entering |
| **Warning** | Create, Update | Inform steward, allow with acknowledgment |
| **Batch Cleansing** | Monthly | Fix existing data gradually |

### 2. Error Message Quality

**BAD:**
```
"Invalid address"
```

**GOOD:**
```
"Address is undeliverable. The postal code 00000 is not valid for Springfield, IL.
Expected format: 62701-1234. Please verify and correct."
```

### 3. Performance Optimization

- Cache DQM results for 24 hours (avoid repeated API calls)
- Batch validate addresses (100 at a time, not one by one)
- Use asynchronous validation for non-blocking scenarios
- Monitor DQM provider API usage and rate limits

### 4. Steward Training

Train data stewards on:
- How to interpret DQM match quality scores
- When to override DQM recommendations
- How to handle partial matches (select correct candidate)
- When to escalate to data governance team
