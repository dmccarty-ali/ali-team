---
name: ali-x12-835-payment
description: |
  X12 835 Health Care Claim Payment/Advice transaction (005010X221A1). Use when:

  PLANNING: Designing payment processing workflows, planning Loop 2100 NM1 entity
  resolution, architecting match rules for payment-to-claim linkage

  IMPLEMENTATION: Parsing 835 files, implementing CLP/CAS segment handlers, building
  match rules for patient/provider/payer entities, creating split tables from NM1_MASTER

  GUIDANCE: Asking about Loop 2100 structure, NM1 entity codes (QC/IL/74/82/TT/PR/DQ),
  payment adjustment codes (CAS segments), crossover/COB scenarios

  REVIEW: Validating 835 parser logic, reviewing entity resolution rules, checking
  split table queries, auditing payment posting workflows

  Do NOT use for shared X12 structure, envelope segments (ISA/GS/ST), BaseParser
  architecture, or section_code hierarchy (use ali-x12-edi-core instead)
---

# X12 835 Health Care Claim Payment/Advice

## Overview

The 835 Health Care Claim Payment/Advice transaction is used by health plans to communicate claim payment and adjustment information to healthcare providers. This skill covers the transaction structure, Loop 2100 NM1 segments, payment processing rules, and integration with match_rules logic.

**Standard:** ASC X12 005010X221A1
**Transaction Set Identifier:** 835
**Purpose:** Report claim payment, denial, or adjustment details

---

## Transaction Structure

```
ISA (Interchange Header)
 └─ GS*HP (Functional Group for Health Care Claims Payment)
     └─ ST*835 (Transaction Set Header)
         ├─ BPR (Financial Information)
         ├─ TRN (Reassociation Trace Number)
         ├─ DTM (Production Date)
         ├─ Loop 1000A (Payer Identification)
         │   ├─ N1*PR (Payer Name)
         │   ├─ N3 (Payer Address)
         │   ├─ N4 (Payer City/State/ZIP)
         │   ├─ REF (Additional Payer ID)
         │   └─ PER (Payer Contact)
         ├─ Loop 1000B (Payee Identification)
         │   ├─ N1*PE (Payee Name)
         │   ├─ N3 (Payee Address)
         │   └─ N4 (Payee City/State/ZIP)
         ├─ Loop 2000 (Provider Summary)
         │   ├─ LX (Header Number)
         │   ├─ TS3 (Provider Summary Info)
         │   └─ Loop 2100 (Claim Payment Info) ← KEY LOOP
         │       ├─ CLP (Claim Payment Information)
         │       ├─ CAS (Claim Adjustment)
         │       ├─ NM1*QC (Patient Name) ← Entity Code QC
         │       ├─ NM1*IL (Insured Name) ← Entity Code IL
         │       ├─ NM1*74 (Corrected Patient) ← Entity Code 74
         │       ├─ NM1*82 (Service Provider) ← Entity Code 82
         │       ├─ NM1*TT (Crossover Carrier) ← Entity Code TT
         │       ├─ NM1*PR (Corrected Payer) ← Entity Code PR
         │       ├─ NM1*DQ (Other Subscriber) ← Entity Code DQ
         │       ├─ MIA (Inpatient Adjudication)
         │       ├─ MOA (Outpatient Adjudication)
         │       ├─ REF (Claim Reference)
         │       ├─ DTM (Claim Dates)
         │       ├─ PER (Claim Contact)
         │       ├─ AMT (Claim Amounts)
         │       ├─ QTY (Claim Quantities)
         │       └─ Loop 2110 (Service Payment Info)
         │           ├─ SVC (Service Payment)
         │           ├─ DTM (Service Date)
         │           ├─ CAS (Service Adjustment)
         │           ├─ REF (Service Identification)
         │           ├─ AMT (Service Amount)
         │           ├─ QTY (Service Quantity)
         │           └─ LQ (Health Care Remark Codes)
         ├─ PLB (Provider Level Adjustment)
         ├─ SE (Transaction Set Trailer)
     └─ GE (Functional Group Trailer)
 └─ IEA (Interchange Trailer)
```

---

## Loop 2100 - Claim Payment Information

### Overview

Loop 2100 contains claim-level payment details and **seven (7) different NM1 segments** that identify parties involved in the claim payment.

**Critical Characteristics:**
- All 7 NM1 segments are at the **same logical position** within Loop 2100
- **Relative order does not matter** per X12 specification
- Each NM1 uses unique **Entity Identifier Code (NM101)** to distinguish party type
- Parser must handle **variable occurrence** (not all 7 appear in every claim)

### Loop 2100 Key Segments

| Segment | Purpose | Usage |
|---------|---------|-------|
| **CLP** | Claim Payment Information | Required (starts loop) |
| **CAS** | Claim Adjustment | Situational (repeats up to 99) |
| **NM1** | Name (7 different entity types) | Required/Situational |
| **MIA** | Inpatient Adjudication | Situational |
| **MOA** | Outpatient Adjudication | Situational |
| **DTM** | Date/Time Reference | Situational |
| **PER** | Claim Contact | Situational |
| **AMT** | Monetary Amount | Situational |
| **QTY** | Quantity | Situational |
| **REF** | Reference Identification | Situational |

---

## Seven NM1 Segments in Loop 2100

### 1. NM1 Patient Name (QC) - REQUIRED

**Entity Identifier Code:** `QC`
**Usage:** **Required** - Must be present in every 835 Loop 2100
**Source:** Original claim (837 Loop 2010CA Patient Name)

**Structure:**
```
NM1*QC*1*SMITH*JOHN*M***MI*123456789A~
```

**Elements:**
- **NM101:** QC (Patient)
- **NM102:** 1=Person, 2=Non-Person
- **NM103:** Patient Last Name (up to 60 chars)
- **NM104:** Patient First Name (up to 35 chars)
- **NM105:** Patient Middle Name (up to 25 chars)
- **NM106:** Name Prefix
- **NM107:** Name Suffix (up to 10 chars)
- **NM108:** MI (Medicare uses MI for HIC#/MBI), also C or other qualifiers
- **NM109:** Patient Identifier (up to 80 chars - HIC#, MBI, Member ID)
- **NM110:** Entity Relationship Code

**Database Table:** `nm1_2100_patient`
**Section Code Pattern:** `section_code ILIKE '%nm1_2100%'`

**Match Rules Usage:**
- Join to claim data via parent_key_hash
- Extract patient identifiers for entity resolution
- Link payments to patient master records

---

### 2. NM1 Insured Name (IL) - SITUATIONAL

**Entity Identifier Code:** `IL`
**Usage:** Situational (Not used by Medicare; used when insured ≠ patient)
**Source:** Original claim (837 Loop 2010BA Subscriber Name)

**Structure:**
```
NM1*IL*1*JONES*MARY*A***MI*987654321B~
```

**When Present:** Insured/subscriber is different from patient (dependent coverage scenarios)

**Database Table:** `nm1_2_2100_insured`
**Section Code Pattern:** `section_code ILIKE '%nm1_2_2100%'`

---

### 3. NM1 Corrected Patient/Insured Name (74) - SITUATIONAL

**Entity Identifier Code:** `74`
**Usage:** Situational (when correcting patient/insured information)

**Structure:**
```
NM1*74*1*SMITH*JOHN*M***C*123456789A~
```

**When Present:** Payer is providing corrected demographic information

**Elements:**
- **NM108:** C (Identification Code Qualifier for corrected info)
- **NM109:** Corrected Insurance Identification Indicator

**Note:** See CMS Attachment 3 with CR 9700 for MBI guidance

**Database Table:** `nm1_3_2100_corrected_patient`
**Section Code Pattern:** `section_code ILIKE '%nm1_3_2100%'`

**Match Rules Usage:**
- Flag discrepancies between claim and payment demographics
- Trigger patient master record updates

---

### 4. NM1 Service Provider Name (82) - SITUATIONAL

**Entity Identifier Code:** `82`
**Usage:** Situational (rendering provider who performed service)

**Structure:**
```
NM1*82*1*DOE*JANE***NPI*1234567890~
```

**When Present:** Identifies specific rendering provider (may differ from billing provider)

**Elements:**
- **NM108:** XX (NPI qualifier) - NPI is standard
- **NM109:** Rendering Provider NPI

**Database Table:** `nm1_4_2100_service_provider`
**Section Code Pattern:** `section_code ILIKE '%nm1_4_2100%'`

**Match Rules Usage:**
- Link payments to provider master records
- Provider-level payment analytics
- NPI validation and provider entity resolution

---

### 5. NM1 Crossover Carrier Name (TT) - SITUATIONAL

**Entity Identifier Code:** `TT`
**Usage:** Situational (coordination of benefits - COB)

**Structure:**
```
NM1*TT*2*MEDICARE PART B***PI*MEDICARE~
```

**When Present:** Payment is being coordinated with another payer (crossover scenario)

**Elements:**
- **NM102:** 2 (Non-Person Entity - payer organization)
- **NM108:** PI or XV (Payer ID qualifier)
- **NM109:** Crossover Carrier Payer ID

**Database Table:** `nm1_5_2100_crossover_carrier`
**Section Code Pattern:** `section_code ILIKE '%nm1_5_2100%'`

**Match Rules Usage:**
- Identify COB scenarios
- Link to secondary payer records
- Track crossover payment chains

---

### 6. NM1 Corrected Priority Payer Name (PR) - SITUATIONAL

**Entity Identifier Code:** `PR`
**Usage:** Situational (correcting payer information)

**Structure:**
```
NM1*PR*2*AETNA***PI*60054~
```

**When Present:** Payer is correcting priority payer identification

**Database Table:** `nm1_6_2100_corrected_payer`
**Section Code Pattern:** `section_code ILIKE '%nm1_6_2100%'`

**Match Rules Usage:**
- Update payer assignments in claims
- Trigger claim resubmission workflows
- Payer entity resolution and correction

---

### 7. NM1 Other Subscriber Name (DQ) - SITUATIONAL

**Entity Identifier Code:** `DQ`
**Usage:** Situational (Not used by Medicare)

**Structure:**
```
NM1*DQ*1*WILLIAMS*ROBERT*J~
```

**When Present:** Additional subscriber information beyond primary insured

**Database Table:** `nm1_7_2100_other_subscriber`
**Section Code Pattern:** `section_code ILIKE '%nm1_7_2100%'`

**Note:** Rarely used in practice; check payer-specific requirements

---

## Entity Identifier Code Quick Reference

| Code | Entity Type | Usage | Medicare | Table Name |
|------|-------------|-------|----------|------------|
| **QC** | Patient Name | Required | Yes | nm1_2100_patient |
| **IL** | Insured Name | Situational | No | nm1_2_2100_insured |
| **74** | Corrected Patient/Insured | Situational | Yes | nm1_3_2100_corrected_patient |
| **82** | Service Provider (Rendering) | Situational | Yes | nm1_4_2100_service_provider |
| **TT** | Crossover Carrier | Situational | Yes | nm1_5_2100_crossover_carrier |
| **PR** | Corrected Priority Payer | Situational | Yes | nm1_6_2100_corrected_payer |
| **DQ** | Other Subscriber | Situational | No | nm1_7_2100_other_subscriber |

---

## Parsing Strategy for Loop 2100 NM1 Segments

### Master + Split Table Pattern

**Master Table:** `NM1_MASTER`
- Contains ALL NM1 segments from all loops (not just 2100)
- Includes `section_code` to identify loop context
- Columns: NM101-NM110, plus metadata (key_hash, parent_key_hash, file_key)

**Split Tables:** One per entity type
```sql
-- Patient Name (QC)
CREATE TABLE nm1_2100_patient AS
SELECT * FROM LAKE_MEDEDI.NM1_MASTER
WHERE section_code ILIKE '%nm1_2100%' AND NM101_ENTITY_ID_CODE = 'QC';

-- Insured Name (IL)
CREATE TABLE nm1_2_2100_insured AS
SELECT * FROM LAKE_MEDEDI.NM1_MASTER
WHERE section_code ILIKE '%nm1_2_2100%' AND NM101_ENTITY_ID_CODE = 'IL';

-- Continue for all 7 entity types...
```

**Why This Works:**
- Single parsing pass writes to master
- Split tables optimized for specific queries
- Easy to add new split tables without re-parsing

---

## Match Rules Integration

### Entity Resolution Patterns

**Pattern 1: Patient Matching**
```sql
-- Match payment to patient master using nm1_2100_patient
UPDATE payment_claim pc
SET patient_id = (
    SELECT p.patient_id
    FROM EDW_MEDEDI.lu_patient p
    INNER JOIN EDW_MEDEDI.nm1_2100_patient nm ON nm.parent_key_hash = pc.key_hash
    WHERE p.patient_identifier = nm.NM109_ID_CODE
      AND p.last_name = nm.NM103_LAST_NAME
)
WHERE pc.patient_id IS NULL;
```

**Pattern 2: Provider Matching**
```sql
-- Match rendering provider using nm1_4_2100_service_provider
UPDATE payment_claim pc
SET rendering_provider_id = (
    SELECT prov.provider_id
    FROM EDW_MEDEDI.lu_provider prov
    INNER JOIN EDW_MEDEDI.nm1_4_2100_service_provider nm ON nm.parent_key_hash = pc.key_hash
    WHERE prov.npi = nm.NM109_ID_CODE
      AND nm.NM108_ID_QUALIFIER = 'XX'
)
WHERE pc.rendering_provider_id IS NULL;
```

**Pattern 3: Crossover/COB Detection**
```sql
-- Flag COB scenarios using nm1_5_2100_crossover_carrier
UPDATE payment_claim pc
SET is_cob = TRUE,
    crossover_payer_id = (
        SELECT payer.payer_id
        FROM EDW_MEDEDI.lu_payer payer
        INNER JOIN EDW_MEDEDI.nm1_5_2100_crossover_carrier nm ON nm.parent_key_hash = pc.key_hash
        WHERE payer.payer_code = nm.NM109_ID_CODE
    )
WHERE EXISTS (
    SELECT 1 FROM EDW_MEDEDI.nm1_5_2100_crossover_carrier nm
    WHERE nm.parent_key_hash = pc.key_hash
);
```

---

## CLP Segment - Claim Payment Information

**Purpose:** Starts Loop 2100 and provides claim-level payment summary

**Structure:**
```
CLP*CLAIM123*1*1000.00*800.00*50.00*12*PATIENTACCT123*22*1~
```

**Key Elements:**
- **CLP01:** Patient Control Number (claim ID from original 837)
- **CLP02:** Claim Status Code
  - 1 = Processed as Primary
  - 2 = Processed as Secondary
  - 3 = Processed as Tertiary
  - 4 = Denied
  - 19 = Processed as Primary, Forwarded to Additional Payer(s)
  - 20 = Processed as Secondary, Forwarded
  - 21 = Processed as Tertiary, Forwarded
  - 22 = Reversal of Previous Payment
  - 23 = Not Our Claim
- **CLP03:** Total Claim Charge Amount
- **CLP04:** Claim Payment Amount
- **CLP05:** Patient Responsibility Amount (copay, coinsurance, deductible)
- **CLP06:** Claim Filing Indicator Code
- **CLP07:** Payer Claim Control Number
- **CLP08:** Facility Type Code
- **CLP09:** Claim Frequency Code

**Match Rules Usage:**
- Link 835 payment to original 837 claim via CLP01
- Reconcile payment amount (CLP04) with expected reimbursement
- Track claim status changes (CLP02)

---

## CAS Segment - Claim Adjustment

**Purpose:** Explains payment adjustments, denials, and reductions

**Structure:**
```
CAS*CO*45*50.00*234*25.00~
```

**Key Elements:**
- **CAS01:** Claim Adjustment Group Code
  - **CO** = Contractual Obligation (not patient responsibility)
  - **OA** = Other Adjustments
  - **PI** = Payer Initiated Reductions
  - **PR** = Patient Responsibility
- **CAS02:** Adjustment Reason Code (CARC - Claim Adjustment Reason Code)
- **CAS03:** Adjustment Amount
- **CAS04:** Adjustment Quantity
- **CAS05-CAS19:** Additional reason/amount pairs (repeats up to 6 times)

**Common Adjustment Reason Codes:**
- **1** = Deductible Amount
- **2** = Coinsurance Amount
- **3** = Copayment Amount
- **45** = Charge exceeds fee schedule/maximum allowable
- **96** = Non-covered charge(s)
- **97** = Payment adjusted because the benefit for this service is included in another service
- **234** = This procedure is not paid separately

**Match Rules Usage:**
- Calculate patient responsibility (CAS01='PR')
- Identify denial reasons
- Trigger appeal workflows for specific CARCs

---

## Payment Processing Flow

### 1. Parse 835 File
```
Parser → NM1_MASTER (all NM1 segments)
      → CLP_MASTER (all claim payment info)
      → CAS_MASTER (all adjustments)
      → Other segment masters
```

### 2. Create Split Tables
```sql
-- Split NM1 by entity type
CREATE TABLE nm1_2100_patient AS SELECT * FROM NM1_MASTER WHERE ...;
CREATE TABLE nm1_4_2100_service_provider AS SELECT * FROM NM1_MASTER WHERE ...;

-- Split CLP to EDW
INSERT INTO EDW_MEDEDI.claim_payment
SELECT clp.*, nm_patient.NM109_ID_CODE as patient_id
FROM CLP_MASTER clp
LEFT JOIN nm1_2100_patient nm_patient ON nm_patient.parent_key_hash = clp.key_hash;
```

### 3. Match Rules Execution
```
1. Match patient (lu_patient)
2. Match provider (lu_provider)
3. Match payer (lu_payer via Loop 1000A)
4. Link to original claim (837)
5. Calculate payment variances
6. Flag exceptions for review
```

### 4. Payment Posting
```
Post payment_amount (CLP04) to accounts receivable
Post patient_responsibility (CLP05) to patient balance
Post adjustments (CAS) to adjustment ledger
Update claim status
```

---

## Field Length Specifications (CMS Flat File)

| Field | Max Length | Notes |
|-------|------------|-------|
| NM103 (Last Name) | 60 | Expanded per HIGLAS |
| NM104 (First Name) | 35 | Expanded per HIGLAS |
| NM105 (Middle Name) | 25 | |
| NM107 (Name Suffix) | 10 | |
| NM109 (ID Code) | 80 | HIC#, MBI, NPI, Payer ID |
| CLP01 (Patient Control #) | 38 | Claim identifier |
| CLP07 (Payer Claim Control #) | 50 | Payer's internal claim # |

**HIGLAS:** Health Insurance Group List Application System

---

## Common Parsing Errors

### Missing Required NM1*QC
**Error:** No patient name in Loop 2100
**Impact:** Cannot process payment - patient identification required
**Solution:** Reject claim payment; flag file for review

### Duplicate Entity Codes
**Error:** Two NM1*QC segments in same Loop 2100
**Impact:** Ambiguous patient identification
**Solution:** Take first occurrence; log warning

### Unknown Entity Code
**Error:** NM1*XX (invalid entity code)
**Impact:** Cannot classify segment
**Solution:** Store in master with warning; manual review

### Missing CLP Segment
**Error:** Loop 2100 without CLP (loop starter)
**Impact:** Cannot identify loop boundary
**Solution:** File-level error; reject entire transaction

---

## Performance Optimization

### Bulk Processing Pattern
```sql
-- Stage all NM1 segments
CREATE TEMP TABLE temp_nm1_batch AS
SELECT * FROM parsed_835_nm1_data;

-- Bulk insert to master
INSERT INTO NM1_MASTER SELECT * FROM temp_nm1_batch;

-- Create all split tables in parallel
CREATE TABLE nm1_2100_patient AS SELECT * FROM NM1_MASTER WHERE section_code ILIKE '%nm1_2100%';
CREATE TABLE nm1_2_2100_insured AS SELECT * FROM NM1_MASTER WHERE section_code ILIKE '%nm1_2_2100%';
-- Continue for all 7...
```

### Indexing Strategy
```sql
-- Master table indexes
CREATE INDEX idx_nm1_master_section ON NM1_MASTER(section_code);
CREATE INDEX idx_nm1_master_entity ON NM1_MASTER(NM101_ENTITY_ID_CODE);
CREATE INDEX idx_nm1_master_parent ON NM1_MASTER(parent_key_hash);

-- Split table indexes
CREATE INDEX idx_nm1_patient_id ON nm1_2100_patient(NM109_ID_CODE);
CREATE INDEX idx_nm1_provider_npi ON nm1_4_2100_service_provider(NM109_ID_CODE);
```

---

## Reference Documentation

**Primary Source:** CMS 835 Flat File Specification (Version 5010A1, Updated 1/10/2023)
**URL:** https://www.cms.gov/medicare/billing/electronicbillingeditrans/downloads/835-flatfile.pdf

**Additional References:**
- [X12 RFI #1724: Multiple NM1 segments in 835](https://x12.org/resources/requests-for-interpretation/rfi-1724-multiple-nm1-segments-835)
- [X12 RFI #2227: Use of NM1*74 on X12 835](https://x12.org/resources/requests-for-interpretation/rfi-2227-use-nm174-x12-835)
- [CMS 2100 Loop Information Bulletin](https://www.cms.gov/files/document/2100-loop-information-bulletin.pdf)
- [Health Care Claim Payment / Advice (835) Companion Guide](https://www.cgsmedicare.com/pdf/edi/835_compguide.pdf)

**Local Reference:** `reference_docs/x12-835-loop-2100-nm1-segments.md`

---

## Related Skills

- **x12-edi-core:** Core X12 fundamentals, envelope structure, parsing patterns
- **x12-837i-institutional:** 837i claims structure, for linking payments back to original claims
- **x12-837p-professional:** 837p claims structure (future)

---

**Document Version:** 1.0
**Last Updated:** 2025-12-14
**Maintained By:** ALI AI Ingestion Engine Team
