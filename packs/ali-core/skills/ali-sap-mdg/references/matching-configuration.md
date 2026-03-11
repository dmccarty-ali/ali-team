# ali-sap-mdg - Matching Configuration Reference

This reference contains detailed matching rule configuration, survivorship logic examples, and DQM integration specifics.

---

## Match Configuration Code Examples

### Deterministic Matching

**Exact match on business keys (Tax ID, Email):**

```abap
* Example: Deterministic match rule in MDG
* Transaction: USMD_RULE
RULE: MATCH_CUSTOMER_TAX_ID
  IF source.TAX_ID = target.TAX_ID
    AND source.TAX_ID IS NOT NULL
  THEN match_score = 100  " Exact match
  ELSE match_score = 0

RULE: MATCH_CUSTOMER_EMAIL
  IF LOWER(source.EMAIL) = LOWER(target.EMAIL)
    AND source.EMAIL IS NOT NULL
  THEN match_score = 100
  ELSE match_score = 0
```

### Probabilistic Matching

**Fuzzy matching with scoring (name similarity, address):**

```abap
RULE: MATCH_CUSTOMER_NAME_ADDRESS
  score = 0

  " Name matching (40 points max)
  IF FUZZY_MATCH(source.NAME, target.NAME) > 0.8
    score = score + 40
  ELSE IF FUZZY_MATCH(source.NAME, target.NAME) > 0.6
    score = score + 20

  " Postal code matching (30 points)
  IF source.POSTAL_CODE = target.POSTAL_CODE
    score = score + 30

  " Street address matching (30 points)
  IF FUZZY_MATCH(source.STREET, target.STREET) > 0.7
    score = score + 30
  ELSE IF FUZZY_MATCH(source.STREET, target.STREET) > 0.5
    score = score + 15

  " Total score determines match probability
  RETURN score  " 0-100
```

### Combined Deterministic + Probabilistic

**Production pattern: Try deterministic first, fall back to probabilistic:**

```abap
RULE: MATCH_CUSTOMER_HYBRID
  " First: Try deterministic matches
  IF source.TAX_ID = target.TAX_ID AND source.TAX_ID IS NOT NULL
    RETURN 100  " Definite match

  IF LOWER(source.EMAIL) = LOWER(target.EMAIL) AND source.EMAIL IS NOT NULL
    RETURN 95  " Very high confidence

  " Second: Probabilistic scoring
  score = 0

  " Name similarity
  name_sim = FUZZY_MATCH(source.NAME, target.NAME)
  IF name_sim > 0.9
    score = score + 50
  ELSE IF name_sim > 0.7
    score = score + 30

  " Address similarity
  IF source.POSTAL_CODE = target.POSTAL_CODE
    score = score + 25

  addr_sim = FUZZY_MATCH(source.STREET, target.STREET)
  IF addr_sim > 0.8
    score = score + 25
  ELSE IF addr_sim > 0.6
    score = score + 15

  RETURN score
```

---

## Survivorship Rules Examples

**Which source wins per attribute when consolidating duplicates:**

### Example 1: Priority-Based Survivorship

```yaml
# Priority-based: Highest priority source wins
customer_name:
  priority:
    1: ERP_GOLD  # Highest priority source
    2: CRM
    3: SALESFORCE
  rule: "Most recent update from highest priority source"

phone_number:
  priority:
    1: CUSTOMER_PORTAL  # Customer-provided is authoritative
    2: SALESFORCE
    3: ERP_GOLD
  rule: "Customer portal overrides all"
```

### Example 2: Attribute-Specific Authority

```yaml
# Different attributes have different authoritative sources
tax_id:
  priority:
    1: GOVERNMENT_REGISTRY  # Only government registry can update
  rule: "Only government registry can update tax ID"
  validation: "Block updates from other sources"

credit_limit:
  priority:
    1: FINANCE_SYSTEM  # Finance system is authoritative
  rule: "Finance system only"
  validation: "Require finance team approval for any change"
```

### Example 3: Data Quality-Based Survivorship

```yaml
# Prefer DQM-validated data over non-validated
address:
  priority:
    1: DQM_VALIDATED  # Data Quality Management validated addresses
    2: CUSTOMER_PORTAL
    3: SALESFORCE
  rule: "Prefer DQM-validated, else most recent"
  logic: |
    IF any_source.DQM_VALIDATED = true
      THEN use highest_priority(sources.where(DQM_VALIDATED = true))
    ELSE use most_recent(sources)

email:
  priority:
    1: VERIFIED_EMAIL  # Email verification system
    2: CUSTOMER_PORTAL
  rule: "Use verified email if available, else most recent"
  validation: "Flag unverified emails for manual review"
```

### Example 4: Time-Based Survivorship

```yaml
# Most recent update wins (no priority)
description:
  rule: "Most recent update wins"
  logic: "MAX(update_timestamp)"

notes:
  rule: "Concatenate all sources (append)"
  logic: "Merge all notes with source attribution"
  format: "[SOURCE] timestamp: note text"
```

### Example 5: Business Rule-Based Survivorship

```yaml
# Complex business logic determines winner
sales_organization:
  rule: "Use org from source that owns the customer"
  logic: |
    IF customer_type = 'NATIONAL_ACCOUNT'
      THEN use CORPORATE_CRM.sales_org
    ELSE IF customer_type = 'REGIONAL'
      THEN use REGIONAL_ERP.sales_org
    ELSE use most_recent(sources.sales_org)

payment_terms:
  rule: "Most restrictive payment terms win"
  logic: |
    winner = MIN(sources.payment_days)  " Shortest payment period
  rationale: "Credit risk mitigation - prefer shorter payment terms"
```

---

## DQM Integration

### Address Validation Configuration

**Transaction:** DQM_MAIN

**Configuration Steps:**

1. **Configure Address Provider:**
   ```
   Provider: Loqate (or Melissa Data, Informatica)
   Connection: HTTPS endpoint
   Authentication: API key (stored in secure storage)
   Countries: US, CA, GB, FR, DE (configure per-country)
   ```

2. **Map MDG Fields to DQM:**
   ```
   MDG Field → DQM Field
   STREET    → AddressLine1
   CITY      → Locality
   STATE     → AdministrativeArea
   POSTAL    → PostalCode
   COUNTRY   → Country
   ```

3. **Define Validation Rules:**
   ```
   Rule: BLOCK_INVALID_ADDRESS
     IF DQM_RESULT.DELIVERABLE = false
     THEN
       BLOCK_CHANGE_REQUEST
       MESSAGE = "Address is undeliverable. Please verify and correct."

   Rule: WARN_PARTIAL_MATCH
     IF DQM_RESULT.MATCH_QUALITY < 0.9
     THEN
       SEND_TO_STEWARD_REVIEW
       MESSAGE = "Address partially validated. Steward review required."

   Rule: AUTO_CORRECT
     IF DQM_RESULT.MATCH_QUALITY >= 0.9
     THEN
       APPLY_STANDARDIZED_ADDRESS
       LOG_CHANGE "Auto-corrected address via DQM"
   ```

4. **Geocoding Configuration:**
   ```
   Enable: Yes
   Update fields: LATITUDE, LONGITUDE
   Use case: Route optimization, territory assignment
   ```

---

## Manual Review Queue

### Process Details for Steward Review

**When matching score is uncertain (70-94%), steward reviews:**

**Review UI (Transaction: NWBC or Fiori app):**
```
+----------------------------------------------------------+
| Manual Match Review: Customer #12345 vs #67890           |
+----------------------------------------------------------+
| Candidate 1          | Candidate 2         | Score: 82%  |
| Acme Corporation     | ACME Corp           |             |
| 123 Main Street      | 123 Main St         |             |
| Anytown, CA 90210    | Anytown CA 90210    |             |
| Tax ID: 12-3456789   | Tax ID: (missing)   |             |
+----------------------------------------------------------+
| [ Merge ]  [ Don't Merge ]  [ Create Rule ]              |
+----------------------------------------------------------+
```

**Steward Actions:**

1. **Merge:** Consolidate the two records
   - System creates golden record using survivorship rules
   - Both source records linked to golden record
   - Change request created for audit trail

2. **Don't Merge:** Keep as separate records
   - System marks as "not a duplicate"
   - Future matching skips this pair (negative rule)
   - Logged for future reference

3. **Create Rule:** Define a new matching/exclusion rule
   - "If Tax ID matches but name differs, always merge"
   - "If postal code differs, never merge"
   - Rule applied to future matches (machine learning)

### Steward Training Topics

**Required training before go-live:**
- Matching concepts (deterministic vs probabilistic)
- Survivorship rule interpretation
- How to spot false positives (different entities, similar data)
- How to spot false negatives (same entity, different data quality)
- When to escalate to data governance team
- How to create effective matching rules

**Best Practice:** Track steward decisions to refine match rules quarterly.

---

## Matching Performance Tuning

### Performance Optimization Techniques

**For large datasets (>100K records):**

1. **Blocking Strategy:**
   ```abap
   " Instead of comparing every record to every other record,
   " create blocks and only compare within blocks

   BLOCK_BY: POSTAL_CODE
   " Only compare customers in same postal code
   " Reduces comparisons from O(n²) to O(n²/b) where b = # of blocks
   ```

2. **Indexing:**
   ```sql
   -- Create database indexes on matching keys
   CREATE INDEX idx_customer_taxid ON BUT000(TAX_ID);
   CREATE INDEX idx_customer_email ON BUT020(EMAIL);
   CREATE INDEX idx_customer_postal ON BUT050(POSTAL_CODE);
   ```

3. **Batch Processing:**
   ```
   Schedule: Nightly matching job (2-4 hour window)
   Chunk size: 10,000 records per batch
   Parallel processing: 4 parallel jobs
   Monitor: Runtime must be <4 hours for daily processing
   ```

4. **Incremental Matching:**
   ```
   Only match new/changed records (delta)
   Don't re-match unchanged records
   Mark: last_match_timestamp per record
   Logic: WHERE update_timestamp > last_match_timestamp
   ```

---

## Matching Quality Metrics

**Track these metrics to tune matching over time:**

| Metric | Target | Action if Off-Target |
|--------|--------|----------------------|
| **False Positive Rate** | <5% | Lower auto-match threshold |
| **False Negative Rate** | <3% | Improve fuzzy match logic |
| **Manual Review Queue Size** | <100/day | Adjust thresholds, add deterministic rules |
| **Steward Decision Time** | <2 min/case | Improve UI, better training |
| **Match Rule Accuracy** | >95% | Refine rules based on steward feedback |
| **Batch Processing Time** | <4 hours | Optimize queries, add indexes, increase parallel jobs |
