---
name: ali-x12-837i-institutional
description: |
  X12 837i Institutional Health Care Claim transaction (005010X223A2). Use when:

  PLANNING: Designing claim ingestion pipelines, planning payer identification
  strategy, architecting Loop 2010BB matching hierarchy

  IMPLEMENTATION: Parsing 837i files, implementing payer matching (NM109/NM103),
  building UB-04 field mappings, handling COB scenarios in Loop 2320

  GUIDANCE: Asking about payer identification best practices, Loop 2010BB structure,
  institutional vs professional differences, claim filing codes

  REVIEW: Validating payer matching logic, reviewing claim structure parsing,
  checking institutional-specific segments (CL1, revenue codes)

  Do NOT use for shared X12 structure, envelope segments (ISA/GS/ST), BaseParser
  architecture, or section_code hierarchy (use ali-x12-edi-core instead)
---

# X12 837i Institutional Health Care Claim

## Overview

The 837i (Institutional) Health Care Claim transaction is used by hospitals, skilled nursing facilities, and other institutional providers to submit claims electronically to payers. This skill covers claim structure, payer identification rules, and institutional-specific requirements.

**Standard:** ASC X12 005010X223A2 (HIPAA 5010)
**Transaction Set Identifier:** 837
**Type:** Institutional (837i)
**Form Equivalent:** UB-04 (CMS-1450)

---

## Transaction Structure

```
ISA (Interchange Header)
 └─ GS*HC (Functional Group for Health Care Claims)
     └─ ST*837 (Transaction Set Header)
         ├─ BHT (Beginning of Hierarchical Transaction)
         ├─ Loop 1000A (Submitter Name)
         │   └─ NM1*41 (Submitter)
         ├─ Loop 1000B (Receiver Name)
         │   └─ NM1*40 (Receiver)
         ├─ Loop 2000A (Billing Provider Hierarchical Level)
         │   ├─ HL (Hierarchical Level)
         │   ├─ Loop 2010AA (Billing Provider Name)
         │   │   ├─ NM1*85 (Billing Provider)
         │   │   ├─ N3 (Billing Provider Address)
         │   │   ├─ N4 (Billing Provider City/State/ZIP)
         │   │   └─ REF (Billing Provider Tax ID, NPI)
         │   └─ Loop 2010AB (Pay-To Provider Name)
         │       └─ NM1*87 (Pay-To Provider)
         ├─ Loop 2000B (Subscriber Hierarchical Level)
         │   ├─ HL (Hierarchical Level)
         │   ├─ SBR (Subscriber Information)
         │   ├─ Loop 2010BA (Subscriber Name)
         │   │   └─ NM1*IL (Subscriber)
         │   └─ Loop 2010BB (Payer Name) ← KEY LOOP FOR PAYER MATCHING
         │       ├─ NM1*PR (Payer)
         │       ├─ N3 (Payer Address)
         │       ├─ N4 (Payer City/State/ZIP)
         │       └─ REF (Payer Secondary ID) [RARELY USED]
         ├─ Loop 2000C (Patient Hierarchical Level)
         │   ├─ HL (Hierarchical Level)
         │   ├─ PAT (Patient Information)
         │   └─ Loop 2010CA (Patient Name)
         │       └─ NM1*QC (Patient)
         ├─ Loop 2300 (Claim Information)
         │   ├─ CLM (Claim Information)
         │   ├─ DTP (Claim Dates)
         │   ├─ CL1 (Institutional Claim Code)
         │   ├─ PWK (Paperwork)
         │   ├─ CN1 (Contract Information)
         │   ├─ AMT (Claim Amounts)
         │   ├─ REF (Claim References)
         │   ├─ HI (Health Care Diagnosis Code) ← Principal and Secondary DX
         │   ├─ HI (Principal Procedure Code) ← Procedures
         │   ├─ Loop 2310A (Attending Provider)
         │   │   └─ NM1*71 (Attending Physician)
         │   ├─ Loop 2310B (Operating Provider)
         │   │   └─ NM1*72 (Operating Physician)
         │   ├─ Loop 2310C (Other Provider)
         │   │   └─ NM1*ZZ (Other Operating Physician)
         │   ├─ Loop 2310D (Rendering Provider)
         │   │   └─ NM1*82 (Rendering Provider)
         │   ├─ Loop 2310E (Service Facility)
         │   │   └─ NM1*77 (Service Facility)
         │   ├─ Loop 2320 (Other Subscriber Information) ← COB
         │   │   ├─ SBR (Other Subscriber)
         │   │   ├─ Loop 2330A (Other Subscriber Name)
         │   │   │   └─ NM1*IL (Other Subscriber)
         │   │   └─ Loop 2330B (Other Payer Name)
         │   │       └─ NM1*PR (Other Payer)
         │   └─ Loop 2400 (Service Line)
         │       ├─ LX (Service Line Number)
         │       ├─ SV2 (Institutional Service Line)
         │       ├─ SVD (Line Adjudication)
         │       ├─ DTP (Service Date)
         │       └─ REF (Service Line References)
         ├─ SE (Transaction Set Trailer)
     └─ GE (Functional Group Trailer)
 └─ IEA (Interchange Trailer)
```

---

## Payer Identification - Loop 2010BB

### Overview

Payer identification is critical for claim routing and processing. Loop 2010BB contains the **primary payer** information and is the most reliable source for payer matching.

**Loop 2010BB Location:**
- Appears within 2000B (Subscriber Hierarchical Level)
- Required loop (must be present in every 837i)
- Contains primary payer responsible for adjudicating the claim

---

## Loop 2010BB - Primary Payer Structure

### NM1 Segment - Payer Name (Required)

```
NM1*PR*2*PAYER_NAME***PI*PAYER_ID~
```

**Standard Elements:**
- **NM101:** Entity Identifier Code = "PR" (Payer)
- **NM102:** Entity Type Qualifier = "2" (Non-Person Entity)
- **NM103:** Payer Name (Last or Organization Name)
- **NM104-NM107:** Not used for payers
- **NM108:** Identification Code Qualifier
  - **PI** = Payer Identification (most common)
  - **XV** = Centers for Medicare & Medicaid Services Plan ID
- **NM109:** Payer Identifier (the actual payer ID)

**Example:**
```
NM1*PR*2*BLUE CROSS BLUE SHIELD***PI*12345~
```

### N3 Segment - Payer Address (Situational)

```
N3*PAYER_ADDRESS_LINE_1*PAYER_ADDRESS_LINE_2~
```

**Usage:** Not typically required for payer matching, but may be present

### N4 Segment - Payer City/State/ZIP (Situational)

```
N4*CITY*ST*ZIP~
```

**Usage:** Geographic information, not typically used for payer matching

### REF Segment - Payer Secondary Identification (Situational)

**CMS NOTE:** Per CMS 837i implementation guide, REF segments in Loop 2010BB are **generally NOT used**. However, the X12 standard allows for:

```
REF*QUALIFIER*IDENTIFIER~
```

**Common Qualifiers (if used):**
- **2U** = Payer Identification Number
- **EI** = Employer's Identification Number (Tax ID)
- **FI** = Federal Taxpayer Identification Number

**CRITICAL:** Most implementations **reject** REF segments in Loop 2010BB. Always check payer-specific companion guides before relying on REF data.

---

## Payer Matching Hierarchy

Use the following priority order to match payers to your system:

### Priority 1: Payer ID with Qualifier (NM108/NM109) ← HIGHEST CONFIDENCE

**Source:** NM109 when NM108 = "PI" or "XV"
**Why First:** Unique, standardized identifier assigned by payer or CMS
**Confidence:** Highest

**Example:**
```
NM1*PR*2*AETNA***PI*60054~
```
**Match using:** Payer ID "60054"

**Match Rule Pattern:**
```sql
SELECT payer_id
FROM EDW_MEDEDI.lu_payer
WHERE payer_code = '60054'
  AND payer_code_qualifier = 'PI';
```

---

### Priority 2: Payer Name (NM103) ← MEDIUM CONFIDENCE

**Source:** NM103 (Payer Name field)
**Why Second:** Fallback when Payer ID is unavailable or unrecognized
**Confidence:** Medium (requires fuzzy matching/normalization)

**Considerations:**
- Name variations exist (e.g., "AETNA", "AETNA HEALTH", "AETNA INC")
- Case sensitivity
- Abbreviations (BC/BS, UHC, etc.)
- Legal suffixes (INC, LLC, CORP)

**Example:**
```
NM1*PR*2*BLUE CROSS BLUE SHIELD OF FLORIDA***PI*UNKNOWN~
```
**Match using:** Name "BLUE CROSS BLUE SHIELD OF FLORIDA"

**Match Rule Pattern:**
```sql
SELECT payer_id
FROM EDW_MEDEDI.lu_payer
WHERE UPPER(TRIM(payer_name)) = UPPER(TRIM('BLUE CROSS BLUE SHIELD OF FLORIDA'));
```

**Name Normalization Best Practices:**
- Convert to uppercase
- Remove extra whitespace
- Expand common abbreviations (BC/BS → BLUE CROSS BLUE SHIELD)
- Strip legal suffixes (INC, LLC, etc.)
- Handle "THE" prefix consistently

---

### Priority 3: Tax ID/EIN (REF with EI/FI qualifier) ← LOW CONFIDENCE

**Source:** REF*EI or REF*FI in Loop 2010BB (rarely used)
**Why Third:** Not standard in Loop 2010BB per CMS; may be rejected
**Confidence:** Low (check payer-specific guides)

**Example (if supported):**
```
REF*EI*591234567~
```
**Match using:** Tax ID "591234567"

**WARNING:** Most clearinghouses and payers reject this approach. Use only if explicitly documented in payer companion guide.

---

### Priority 4: Combination Matching ← LAST RESORT

When single identifiers fail, use combinations:
- **Payer Name + State** (from N4 segment)
- **Payer Name + Claim Filing Code** (from SBR09)

**Match Rule Pattern:**
```sql
SELECT payer_id
FROM EDW_MEDEDI.lu_payer
WHERE UPPER(payer_name) LIKE '%BLUE CROSS%'
  AND payer_state = 'FL'
  AND claim_filing_code = '12';  -- PPO
```

---

## Loop 2320 - Other Payer Structure (COB)

### Purpose

Used for **Coordination of Benefits** (COB) when multiple payers are involved (secondary, tertiary, etc.).

### SBR Segment - Other Subscriber Information (Required)

```
SBR*SEQUENCE*RELATIONSHIP*GROUP_NO*GROUP_NAME*INSURANCE_TYPE****CLAIM_FILING_CODE~
```

**Key Elements:**
- **SBR01:** Payer Responsibility Sequence Number Code
  - **P** = Primary
  - **S** = Secondary
  - **T** = Tertiary
- **SBR09:** Claim Filing Indicator Code (identifies payer type)
  - **12** = Preferred Provider Organization (PPO)
  - **13** = Point of Service (POS)
  - **14** = Exclusive Provider Organization (EPO)
  - **15** = Indemnity Insurance
  - **16** = Health Maintenance Organization (HMO) Medicare Risk
  - **MA** = Medicare Part A
  - **MB** = Medicare Part B
  - **MC** = Medicaid
  - **CH** = CHAMPUS (TRICARE)
  - **VA** = Veterans Affairs Plan

### Loop 2330B - Other Payer Name (Required)

Contains NM1 segment similar to Loop 2010BB structure:

```
NM1*PR*2*OTHER_PAYER_NAME***PI*OTHER_PAYER_ID~
```

**Same matching rules apply as Loop 2010BB**

---

## Payer Identification Validation Rules

### 1. Required Fields Check

- **NM101** must equal "PR"
- **NM102** must equal "2"
- **NM103** must be present (payer name cannot be empty)
- **NM108/NM109** should be present (check payer requirements)

### 2. Qualifier Validation

- **NM108** should be "PI" or "XV" (per CMS standards)
- Other qualifiers may cause claim rejection

### 3. Format Validation

- **Payer ID (NM109):** Alphanumeric, typically 5-10 characters
- **Payer Name (NM103):** Up to 60 characters
- No special validation required for name field (free text)

### 4. Cross-Loop Consistency

- Primary payer (Loop 2010BB) must be present
- If Loop 2320 exists, it should reference different payers
- SBR01 sequence (P, S, T) should align with loop hierarchy

---

## Common Payer Identification Scenarios

### Scenario 1: Standard Medicare Claim

```
NM1*PR*2*MEDICARE***PI*MEDICARE~
```

**Match on:** Name "MEDICARE" or Payer ID "MEDICARE"
**Notes:** Medicare uses various formats; normalize consistently

### Scenario 2: Commercial Payer with Payer ID

```
NM1*PR*2*UNITED HEALTHCARE***PI*87726~
```

**Match on:** Payer ID "87726" (highest confidence)
**Fallback:** Name "UNITED HEALTHCARE"

### Scenario 3: Payer with No ID

```
NM1*PR*2*LOCAL HEALTH PLAN**~
```

**Match on:** Name "LOCAL HEALTH PLAN" only
**Note:** Missing NM108/NM109; lower confidence matching

### Scenario 4: COB with Multiple Payers

```
Loop 2010BB (Primary):
NM1*PR*2*BLUE CROSS***PI*12345~

Loop 2320 (Secondary):
SBR*S*18*GROUP123****12~
NM1*PR*2*AETNA***PI*60054~

Loop 2320 (Tertiary):
SBR*T*18*GROUP456****13~
NM1*PR*2*CIGNA***PI*62308~
```

**Process:**
1. Primary: Blue Cross (Payer ID 12345)
2. Secondary: Aetna (Payer ID 60054)
3. Tertiary: Cigna (Payer ID 62308)

---

## Institutional Claim Specifics

### CL1 Segment - Institutional Claim Code

**Purpose:** Captures UB-04 claim codes specific to institutional billing

```
CL1*1*9*01~
```

**Key Elements:**
- **CL1 01:** Admission Type Code (UB-04 FL 14)
- **CL1 02:** Admission Source Code (UB-04 FL 15)
- **CL1 03:** Patient Status Code (UB-04 FL 17)

### Diagnosis Coding (HI Segments)

**Multiple HI segments for diagnosis:**

```
HI*ABK:I10~               ← Principal Diagnosis (ABK qualifier)
HI*ABF:E119*ABF:I10~      ← Secondary Diagnoses (ABF qualifier)
```

**ICD-10 Qualifiers:**
- **ABK** = Principal Diagnosis
- **ABF** = Other Diagnosis

### Procedure Coding (HI Segments)

```
HI*BBR:0DB43ZX~           ← Principal Procedure (BBR qualifier)
HI*BBQ:0DH60MZ~           ← Other Procedures (BBQ qualifier)
```

**ICD-10-PCS Qualifiers:**
- **BBR** = Principal Procedure
- **BBQ** = Other Procedure

### Revenue Codes (SV2 Segment)

**Service Line for Institutional Claims:**

```
SV2*0450*HC:99283*75.00*UN*1~
```

**Key Elements:**
- **SV201:** Revenue Code (UB-04 FL 42)
- **SV202:** HCPCS Code (procedure)
- **SV203:** Line Item Charge Amount
- **SV204:** Unit Basis Code (UN=Unit)
- **SV205:** Service Unit Count

**Common Revenue Codes:**
- **0450** = Emergency Room
- **0300** = Laboratory
- **0258** = Intravenous Therapy
- **0636** = Drugs - Single Source

---

## UB-04 Form Field Mapping

| UB-04 Field | X12 Segment/Element | Description |
|-------------|---------------------|-------------|
| FL 1 | NM1*77 Loop 2310E | Service Facility Name |
| FL 3a-3b | Loop 2000C PAT | Patient Birth Date, Sex |
| FL 4 | CLM05-3 | Type of Bill |
| FL 5 | N4 Loop 2010AA | Billing Provider Address |
| FL 6 | DTP*435 | Statement Covers From Date |
| FL 12-17 | CL1 | Admission/Discharge Info |
| FL 31-36 | Loop 2300 CLM | Occurrence Codes/Dates |
| FL 38 | NM1*71 Loop 2310A | Attending Provider |
| FL 39-41 | Loop 2300 HI | Value Codes and Amounts |
| FL 42 | SV2*01 | Revenue Code |
| FL 44 | SV2*03 | HCPCS/Procedure Code |
| FL 45 | Loop 2310D NM1*82 | Rendering Provider |
| FL 46 | SV2*05 | Service Units |
| FL 47 | SV2*04 | Total Charges |
| FL 50 | NM1*PR Loop 2010BB | **Primary Payer Name** |
| FL 56 | NPI in Loop 2010AA | **Billing Provider NPI** |
| FL 57 | REF*EI Loop 2010AA | **Billing Provider Tax ID** |
| FL 60 | Patient Control # (CLM01) | Insured Unique ID |
| FL 61 | DTP Loop 2300 | Insured Group Name/Number |
| FL 67 | Loop 2330B NM1*PR | Other Payer Name (COB) |

---

## Implementation Considerations

### Handling Unknown Payers

When payer cannot be matched to your system:

1. **Extract all available identifiers:**
   - Payer ID (NM109)
   - Payer Name (NM103)
   - Qualifier type (NM108)

2. **Flag for manual review** with context:
   - Full NM1 segment
   - Claim Filing Code (from SBR09)
   - Any address information (N3/N4)

3. **Create temporary payer record** to allow claim processing

4. **Link to master payer** after manual resolution

### Multiple Payers (COB Scenarios)

Process in sequence order:
1. Parse Loop 2010BB (primary payer)
2. Parse all Loop 2320 iterations (other payers)
3. Order by SBR01 (P, S, T)
4. Match each payer independently
5. Store coordination of benefits linkage

---

## Error Handling

### Invalid Payer Segments

**Missing NM1 in Loop 2010BB:**
- **Action:** Reject claim at file level
- **Reason:** Primary payer is required per HIPAA standard

**Invalid Entity Identifier (NM101 ≠ "PR"):**
- **Action:** Reject claim
- **Reason:** Loop 2010BB must identify a payer

**Invalid Entity Type (NM102 ≠ "2"):**
- **Action:** Warning/rejection (check payer rules)
- **Reason:** Payers should be non-person entities

**Missing Payer Name (NM103):**
- **Action:** Reject claim
- **Reason:** Payer name is required for matching

### Unmatched Payers

When payer cannot be matched:
1. Log all extracted identifiers for analysis
2. Flag claim for manual payer assignment
3. Generate reports of unmatched payers for master data updates
4. Track unmatched frequency to identify missing payers in system

---

## Match Rules SQL Patterns

### Pattern 1: Payer ID Match (Highest Priority)

```sql
UPDATE claim c
SET payer_id = (
    SELECT p.payer_id
    FROM EDW_MEDEDI.lu_payer p
    INNER JOIN EDW_MEDEDI.nm1_2010bb_payer nm ON nm.parent_key_hash = c.subscriber_key_hash
    WHERE p.payer_code = nm.NM109_ID_CODE
      AND p.payer_code_qualifier = nm.NM108_ID_QUALIFIER
      AND nm.NM108_ID_QUALIFIER IN ('PI', 'XV')
)
WHERE c.payer_id IS NULL;
```

### Pattern 2: Payer Name Match (Fallback)

```sql
UPDATE claim c
SET payer_id = (
    SELECT p.payer_id
    FROM EDW_MEDEDI.lu_payer p
    INNER JOIN EDW_MEDEDI.nm1_2010bb_payer nm ON nm.parent_key_hash = c.subscriber_key_hash
    WHERE UPPER(TRIM(p.payer_name)) = UPPER(TRIM(nm.NM103_PAYER_NAME))
)
WHERE c.payer_id IS NULL;
```

### Pattern 3: COB Payer Matching

```sql
-- Match secondary payer from Loop 2320/2330B
UPDATE claim c
SET secondary_payer_id = (
    SELECT p.payer_id
    FROM EDW_MEDEDI.lu_payer p
    INNER JOIN EDW_MEDEDI.nm1_2330b_other_payer nm ON nm.parent_key_hash = c.key_hash
    WHERE p.payer_code = nm.NM109_ID_CODE
      AND EXISTS (
          SELECT 1 FROM EDW_MEDEDI.sbr_other_subscriber sbr
          WHERE sbr.parent_key_hash = c.key_hash
            AND sbr.SBR01_PAYER_SEQUENCE = 'S'  -- Secondary
      )
)
WHERE c.secondary_payer_id IS NULL;
```

---

## Reference Standards

**Primary Source:** ASC X12 005010X223A2 Implementation Guide
**CMS Guidance:** CMS 837i Companion Guide (Transaction Instructions)
**HIPAA Regulations:** 45 CFR 162 (Administrative Simplification)

**Note:** Always consult payer-specific companion guides for implementation variations. While the X12 standard provides the structure, individual payers may have specific requirements for identifiers, formats, and validation rules.

---

## Related Skills

- **x12-edi-core:** Core X12 fundamentals, envelope structure, parsing patterns
- **x12-835-payment:** 835 Payment transaction for linking payments back to 837i claims
- **x12-837p-professional:** 837p Professional Claims (future)

---

**Document Version:** 1.0
**Last Updated:** 2025-12-14
**Maintained By:** ALI AI Ingestion Engine Team
