# ali-data-architecture - Naming Conventions Reference

Standard abbreviations, prefixes, and suffixes for database objects and columns.

---

## Standard Abbreviations

Approved abbreviations for database object and column names. Use these consistently across all schemas.

| Abbrev | Full Word | Example |
|--------|-----------|---------|
| ABBR | Abbreviation | claim_status_abbr |
| ACCT | Account | sender_bank_acct_num |
| ACK | Acknowledgment | ack_req |
| ADDL | Additional | addl_data |
| ADDR | Address | addr_info |
| ADJ | Adjustment | adj_reason_code |
| ADMIN | Administrator | admin_name |
| ALT | Alternate | alt_id |
| AMT | Amount | monetary_amt |
| APP | Application | app_sender_code |
| ASSGN | Assignment | benefits_assgn_ind |
| AUTH | Authorization | auth_number |
| AVAIL | Availability | non_avail_cert |
| BEN | Benefits | benefits_assgn_ind |
| CERT | Certificate | certification_date |
| CHRG | Charge | total_claim_chrg |
| CLM | Claim | clm_2300_claim_header |
| CLP | Claim Payment | clp_claim_payment_master |
| COMM | Communication | comm_num_qual |
| COND | Condition | yes_no_cond_resp |
| COORD | Coordination | coord_ben_priority |
| CRED | Credit/Credential | cred_deb_flag_code |
| CTRL | Control | intrchg_ctrl_num |
| CUR | Currency | cur_currency_master |
| DEB | Debit | cred_deb_flag_code |
| DEP | Deposit | dep_fin_inst |
| DESC | Description | claim_status_desc |
| DEST | Destination | dest_tbl |
| DFI | Depository Financial Institution | dfi_id_num |
| DIAG | Diagnosis | hi_diagnosis_master |
| DRG | Diagnosis Related Group | drg_master |
| DTL | Detail | cas_adjustment_detail |
| DUP | Duplicate | dup_clm_match |
| DTM | Date/Time | dtm_dates_master |
| DTP | Date Time Period | dtp_date_time_period_master |
| EFF | Effective | plan_eff_date |
| EFT | Electronic Funds Transfer | eft_effective_date |
| EIN | Employer ID Number | employer_ein |
| ENC | Encounter | claim_or_enc_id_code |
| ESRD | End Stage Renal Disease | esrd_payment_amt |
| EXCH | Exchange | exch_rate |
| EXP | Expiration | plan_exp_date |
| FAC | Facility | fac_type_code |
| FCT | Function | fctl_id_code |
| FCTL | Functional | functional_group_trailer |
| FIN | Financial | fin_inst |
| FREQ | Frequency | claim_freq_id |
| GRP | Group | functional_group |
| HCPCS | Healthcare Common Procedure Coding | hcpcs_code |
| HIER | Hierarchical | hier_lvl_code |
| HOSP | Hospital | hosp_name |
| ID | Identifier | payer_id |
| IDX | Index | loop_idx |
| IME | Indirect Medical Education | ime_amt |
| IND | Indicator | provider_sig_ind |
| INCL | Included | num_incl_fctl_grps |
| INFO | Information | addr_info |
| INST | Institution | fin_inst |
| INTRCHG | Interchange | intrchg_ctrl_num |
| LKUP | Lookup | lkup_tbl |
| LOC | Location | hosp_loc |
| LQ | Loop Qualifier | lq_form_id_master |
| LU | Lookup (prefix) | lu_claim_status |
| LVL | Level | lvl_of_care_code |
| MEAS | Measurement | unit_basis_meas_code |
| MIA | Medicare Inpatient Adjustment | mia_master |
| MOA | Medicare Outpatient Adjustment | moa_master |
| MONET | Monetary | monetary_amt |
| MRKT | Market | cur_mrkt_exch_code |
| MSG | Message | error_msg |
| MSP | Medicare Secondary Payer | msp_pass_through_amt |
| NATL | National | natl_drug_unit_count |
| NM | Name (X12 segment) | nm1_master |
| NPI | National Provider Identifier | npi |
| NUM | Number | contract_num |
| OID | Object Identifier | coding_sys_oid |
| ORD | Ordinal | ordinal_position |
| ORG | Organization | org_name |
| ORIG | Original/Originator | orig_co_supp_code |
| PAT | Patient | pat_patient_info_master |
| PER | Contact Person | per_master |
| PLB | Provider Level Balance | plb_provider_id |
| POS | Position | ordinal_position |
| PPS | Prospective Payment System | pps_capital_amt |
| PREV | Previous | prev_value |
| PROC | Procedure | proc_code_id |
| PROD | Product | prod_svc_qualifier |
| PROF | Professional | prof_comp_amt |
| PROV | Provider | prov_id |
| PYMT | Payment | claim_pymt_status |
| QTY | Quantity | quantity |
| QUAL | Qualifier | date_format_qual |
| RCVR | Receiver | intrchg_rcvr_id |
| REAS | Reason | adj_reason_code |
| REF | Reference | ref_2100 |
| REIMB | Reimbursement | reimb_rate |
| REL | Relationship | entity_rel_code |
| REQ | Request | ack_req |
| RESP | Response | yes_no_cond_resp |
| RMK | Remark | lq_rmk_code |
| RX | Prescription | rx_num |
| SEC | Security/Section | sec_info_qual |
| SEP | Separator | component_elem_sep |
| SEQ | Sequence | adjustment_seq |
| SIG | Signature | provider_sig_ind |
| SK | Surrogate Key | patient_sk |
| SPEC | Specific/Specialist | hosp_spec_drg_amt |
| SRC | Source | src_file |
| SSN | Social Security Number | insured_ssn |
| STG | Staging | sk_stg |
| STMT | Statement | sql_stmts |
| SUBM | Submission | subm_code |
| SUPP | Supplemental | svc_supp_amt |
| SVC | Service | svc_2110 |
| SYS | System | sys_control |
| TBL | Table | lkup_tbl |
| TIN | Tax ID Number | org_tin |
| TS | Timestamp | event_ts |
| TXN | Transaction | txn_handling_code |
| TXT | Text | note_txt |
| UOM | Unit of Measure | uom_code |
| VER | Version | ver_rel_industry_id |
| XREF | Cross-Reference | xref_claim_payer_master |

---

## Table Prefixes

| Prefix | Purpose | Example |
|--------|---------|---------|
| **LU_** | Lookup/reference table | LU_CLAIM_STATUS |
| **XREF_** | Cross-reference/mapping table | XREF_CLAIM_PAYER_MASTER |
| **_MASTER** | Contains all occurrences (X12 term) | NM1_MASTER |
| **_DETAIL** | Child/detail records | CAS_ADJUSTMENT_DETAIL |
| **DIM_** | Dimension table (data warehouse) | DIM_PROVIDER |
| **FACT_** | Fact table (data warehouse) | FACT_CLAIM_LINE |
| **STG_** | Staging layer (silver) | STG_MEDEDI.CLAIM |
| **LAKE_** | Raw layer (bronze) | LAKE_MEDEDI.EDI_837I_RAW |
| **EDW_** | Enterprise data warehouse (gold) | EDW_MEDEDI.FACT_CLAIM |
| **TMP_** | Temporary table | TMP_CLAIM_PROCESSING |
| **BAK_** | Backup table | BAK_CLAIM_20260115 |
| **ARC_** | Archive table | ARC_CLAIM_2025 |
| **SYS_** | System/control table | SYS_CONTROL.LOAD_WATERMARKS |

---

## Column Suffixes

| Suffix | Purpose | Data Type | Example |
|--------|---------|-----------|---------|
| **_SK** | Surrogate key (FK to dimension) | INTEGER | payer_sk |
| **_ID** | Natural identifier | VARCHAR | payer_id |
| **_CODE** | Code value (typically FK to lookup) | VARCHAR | claim_status_code |
| **_DESC** | Description (human-readable) | VARCHAR | claim_status_desc |
| **_FLAG** | Boolean indicator | BOOLEAN | active_flag |
| **_IND** | Indicator (Y/N, 0/1) | VARCHAR(1) or BOOLEAN | provider_sig_ind |
| **_TS** | Timestamp (date + time) | TIMESTAMP_NTZ | event_ts |
| **_DATE** | Date only (no time) | DATE | service_date |
| **_TIME** | Time only (no date) | TIME | arrival_time |
| **_AMT** | Monetary amount | NUMBER(12,2) | charge_amt |
| **_QTY** | Quantity/count | INTEGER | unit_qty |
| **_NUM** | Number (non-monetary) | INTEGER or VARCHAR | contract_num |
| **_PCT** | Percentage | NUMBER(5,2) | tax_pct |
| **_RATE** | Rate/ratio | NUMBER(8,4) | interest_rate |
| **_KEY** | Composite/hash key | VARCHAR(64) | claim_key (MD5) |
| **_JSON** | JSON blob | VARIANT or TEXT | metadata_json |
| **_ARRAY** | Array type | ARRAY | diagnosis_codes_array |

---

## Schema Naming Patterns

| Schema Pattern | Purpose | Example |
|----------------|---------|---------|
| **LAKE_{domain}** | Raw ingestion layer | LAKE_MEDEDI, LAKE_API |
| **STG_{domain}** | Staging/cleaned layer | STG_MEDEDI, STG_CRM |
| **EDW_{domain}** | Enterprise data warehouse | EDW_MEDEDI, EDW_FINANCE |
| **SYS_CONTROL** | System metadata/control | SYS_CONTROL (watermarks, logs) |
| **SANDBOX_{user}** | Personal development | SANDBOX_JSMITH |
| **ARC_{domain}** | Archive schemas | ARC_MEDEDI |

---

## Naming Best Practices

### General Rules

1. **Use UPPERCASE for schema and table names**
   - Schema: `STG_MEDEDI`
   - Table: `CLAIM`
   - Full: `STG_MEDEDI.CLAIM`

2. **Use snake_case for column names**
   - Column: `claim_status_code`
   - Not: `ClaimStatusCode` or `claimStatusCode`

3. **Be consistent with abbreviations**
   - Use `AMT` everywhere for Amount
   - Don't mix `AMT` and `AMOUNT` in same schema

4. **Avoid reserved words**
   - Don't name columns: `date`, `order`, `group`, `user`
   - Instead: `service_date`, `order_num`, `group_id`, `user_id`

5. **Use singular for table names**
   - Table: `CLAIM` (not `CLAIMS`)
   - Exception: When plural is the domain term (BENEFITS, SERVICES)

6. **Make names self-documenting**
   - Good: `claim_submission_date`
   - Bad: `dt1` or `sd`

### Dimension Tables

```
DIM_{entity}
```

Examples:
- `DIM_PROVIDER` - Provider dimension
- `DIM_PATIENT` - Patient dimension
- `DIM_DATE` - Date dimension
- `DIM_PROCEDURE` - Procedure code dimension

### Fact Tables

```
FACT_{grain}
```

Examples:
- `FACT_CLAIM` - One row per claim
- `FACT_CLAIM_LINE` - One row per claim line item
- `FACT_PAYMENT` - One row per payment transaction
- `FACT_DAILY_SUMMARY` - One row per day per provider

### Bridge Tables (Many-to-Many)

```
BRIDGE_{entity1}_{entity2}
```

Examples:
- `BRIDGE_CLAIM_DIAGNOSIS` - Claim to diagnosis codes
- `BRIDGE_PROVIDER_NETWORK` - Provider to network
- `BRIDGE_PATIENT_PAYER` - Patient to payer (COB)

### Audit Columns (Standard Set)

Every table should have these:

```sql
-- Who and when loaded
load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
loaded_by           VARCHAR(100) DEFAULT CURRENT_USER,

-- Source tracking
source_system       VARCHAR(50),
source_file_id      VARCHAR(100),
source_record_id    VARCHAR(100),

-- Batch tracking
etl_batch_id        VARCHAR(100),

-- Version tracking (optional)
record_version      INTEGER DEFAULT 1,
row_hash            VARCHAR(64)  -- MD5 of content for change detection
```

### SCD Type 2 Columns (Standard Set)

For dimensions tracking history:

```sql
-- Surrogate key
{entity}_sk         INTEGER PRIMARY KEY,

-- Natural key
{entity}_id         VARCHAR(50) NOT NULL,

-- SCD Type 2 tracking
valid_from          DATE NOT NULL,
valid_to            DATE NOT NULL DEFAULT '9999-12-31',
is_current          BOOLEAN NOT NULL DEFAULT TRUE,

-- Audit
load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP
```

---

## Examples by Domain

### Healthcare Claims

```sql
-- Tables
STG_MEDEDI.CLAIM
STG_MEDEDI.CLAIM_LINE
EDW_MEDEDI.FACT_CLAIM
EDW_MEDEDI.DIM_PROVIDER
LU_CLAIM_STATUS

-- Columns
claim_id
claim_line_num
claim_status_code
claim_status_desc
provider_npi
provider_sk
service_date
charge_amt
paid_amt
denial_flag
```

### Finance/Accounting

```sql
-- Tables
STG_FINANCE.TRANSACTION
EDW_FINANCE.FACT_PAYMENT
EDW_FINANCE.DIM_ACCOUNT
LU_TXN_TYPE

-- Columns
txn_id
txn_type_code
acct_num
debit_amt
credit_amt
txn_ts
balance_amt
```

### Customer/CRM

```sql
-- Tables
STG_CRM.CUSTOMER
STG_CRM.INTERACTION
EDW_CRM.DIM_CUSTOMER
EDW_CRM.FACT_INTERACTION

-- Columns
customer_id
customer_sk
first_name
last_name
email_addr
phone_num
interaction_date
interaction_type_code
```

---

## Abbreviation Conflicts (Resolve with Context)

When abbreviation could mean multiple things, add context:

| Ambiguous | Clarified |
|-----------|-----------|
| DATE | service_date vs submission_date vs load_date |
| STATUS | claim_status vs payment_status vs patient_status |
| TYPE | claim_type vs patient_type vs provider_type |
| CODE | diagnosis_code vs procedure_code vs status_code |
| NAME | provider_name vs patient_name vs org_name |
| ID | claim_id vs provider_id vs patient_id |

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why Bad | Do Instead |
|--------------|---------|------------|
| `tbl_claim` | Redundant prefix | Just `CLAIM` |
| `claims_table` | Redundant suffix | Just `CLAIM` |
| `ClaimStatusCode` | CamelCase inconsistent | `claim_status_code` |
| `clm_sts_cd` | Over-abbreviated | `claim_status_code` |
| `date_1`, `date_2` | Non-descriptive | `service_date`, `submission_date` |
| `field_123` | Meaningless | Describe what it contains |
| `temp`, `tmp1`, `tmp2` | No context | `TMP_CLAIM_DEDUP_WORK` |
| `col1`, `col2` | Generic | Use actual meaning |
| `flag` without prefix | Ambiguous | `active_flag`, `deleted_flag` |

---

## Decision Tree for Naming

```
Is it a table?
├─ Is it a dimension? → DIM_{entity}
├─ Is it a fact? → FACT_{grain}
├─ Is it a lookup? → LU_{entity}
├─ Is it a cross-reference? → XREF_{entity1}_{entity2}
└─ Is it staging? → Use layer prefix: STG_, LAKE_, EDW_

Is it a column?
├─ Is it a surrogate key? → {entity}_sk
├─ Is it a natural ID? → {entity}_id
├─ Is it a code? → {entity}_{attribute}_code
├─ Is it a description? → {entity}_{attribute}_desc
├─ Is it a boolean? → {attribute}_flag or {attribute}_ind
├─ Is it a date/time? → {event}_date or {event}_ts
├─ Is it an amount? → {type}_amt
└─ Is it a quantity? → {item}_qty
```

---

## Glossary Integration

Maintain a data dictionary mapping:

```yaml
# data_dictionary.yaml
terms:
  claim_id:
    definition: "Unique identifier for a healthcare claim"
    domain: "Healthcare"
    data_type: "VARCHAR(50)"
    pattern: "^CLM[0-9]{10}$"
    source_system: "Claims Processing System"

  provider_npi:
    definition: "National Provider Identifier (10-digit)"
    domain: "Healthcare"
    data_type: "VARCHAR(10)"
    pattern: "^[0-9]{10}$"
    source_system: "NPI Registry"
    standard: "CMS NPI"
```

---

## Summary

- **Abbreviations**: Use approved list consistently
- **Prefixes**: Indicate table purpose (DIM_, FACT_, LU_, XREF_)
- **Suffixes**: Indicate column semantics (_SK, _ID, _CODE, _DESC, _FLAG, _AMT)
- **Case**: UPPERCASE for schemas/tables, snake_case for columns
- **Audit columns**: Standardize across all tables
- **Documentation**: Maintain data dictionary with definitions
