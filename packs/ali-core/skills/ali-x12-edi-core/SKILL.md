---
name: ali-x12-edi-core
description: |
  Core X12 EDI fundamentals and patterns shared across all healthcare transactions
  (835, 837i, 837p). Use when:

  PLANNING: Designing X12 parser architecture, planning master+split table patterns,
  architecting section_code hierarchies, choosing parse context strategies

  IMPLEMENTATION: Building BaseParser templates, implementing segment handlers,
  creating X12Segment classes, writing key_hash generation logic, building
  RowAccumulator flush patterns

  GUIDANCE: Asking about ISA fixed-length requirements, envelope structure, NM1
  entity codes, delimiter handling, micro-partition strategies, qualifier-based
  segment lookup

  REVIEW: Validating parser architecture, reviewing parent-child key linking,
  checking QUALIFY deduplication logic, auditing flush threshold configurations

  Do NOT use for:
  - Transaction-specific loops (use ali-x12-835-payment for Loop 2100, ali-x12-837i-institutional for Loop 2010BB)
  - Payment adjustment codes (use ali-x12-835-payment)
  - Claim filing codes or institutional-specific segments (use ali-x12-837i-institutional)
  - Payer identification rules (use ali-x12-837i-institutional)
  - Transaction-specific entity codes beyond the shared fundamentals listed in Entity Identifier Codes table
---

# X12 EDI Core Fundamentals

## Overview

This skill provides foundational knowledge for all X12 HIPAA healthcare transactions. It covers the core structure, envelope segments, parsing patterns, and architectural concepts that are shared across 835 (Payment), 837i (Institutional Claims), 837p (Professional Claims), and other healthcare EDI transactions.

## X12 Transaction Structure

### Envelope Hierarchy

X12 transactions use a nested envelope structure:

```
ISA (Interchange Control Header)
 └─ GS (Functional Group Header)
     └─ ST (Transaction Set Header)
         └─ [Transaction-Specific Segments]
             └─ Loops (repeating segments)
                 └─ Segments (data elements)
         └─ SE (Transaction Set Trailer)
     └─ GE (Functional Group Trailer)
 └─ IEA (Interchange Control Trailer)
```

### Envelope Segments

#### ISA - Interchange Control Header
**Purpose:** Outermost envelope; defines sender/receiver and interchange-level control

**Key Elements:**
- ISA05/06: Interchange Sender ID Qualifier and ID
- ISA07/08: Interchange Receiver ID Qualifier and ID
- ISA09: Interchange Date (YYMMDD)
- ISA10: Interchange Time (HHMM)
- ISA13: Interchange Control Number (unique per interchange)
- ISA15: Usage Indicator (P=Production, T=Test)
- ISA16: Component Element Separator

**Example:**
```
ISA*00*          *00*          *ZZ*SENDER123      *ZZ*RECEIVER456    *201231*1234*^*00501*000000001*0*P*:~
```

**CRITICAL: ISA Fixed-Length Requirement**

The ISA segment is unique in X12 - it has **fixed-length elements** (unlike all other segments which are variable-length delimited). This is because the ISA defines what the delimiters ARE, so parsers cannot use delimiters to parse it.

**The ISA segment is EXACTLY 106 characters.**

| Element | Length | Description |
|---------|--------|-------------|
| ISA | 3 | Segment ID |
| Separators | 16 | Element separators (one per position) |
| Data | 86 | Fixed-length data fields |
| Terminator | 1 | Segment terminator |
| **Total** | **106** | |

**Element Length Requirements:**
- ISA02, ISA04: 10 characters (right-padded with spaces)
- ISA06, ISA08: 15 characters (right-padded with spaces)
- ISA13: 9 characters (left-padded with zeros)

**Delimiter Positions (0-indexed):**
- Position 3: Element separator (typically `*`)
- Position 104: Repetition separator (ISA11)
- Position 105: Component separator (typically `:`)
- Position 106: Segment terminator (typically `~`)

**Parser Implementation:**
- Read EXACTLY 106 characters - no more, no less
- Extract delimiters from fixed positions
- If file has non-standard ISA length, it is malformed and should fail parsing

**Why This Matters:**
- Parsers rely on the exact 106-character length to locate delimiters
- Reading more than 106 characters will consume the beginning of the GS segment
- Reading fewer than 106 characters will miss delimiter detection
- There are NO industry variations - files with wrong ISA length are invalid

**Sources:**
- [Stedi - X12 EDI ISA Interchange Control Header](https://www.stedi.com/edi/x12/segment/ISA)
- [CMS 1500 - EDI ISA Segment Fixed Length](https://cms1500claimbilling.com/isa-interchange-control-header-segment/)
- [RDPCrystal - Detecting X12 EDI Delimiters](https://www.rdpcrystal.com/detecting-edi-delimiters/)

#### GS - Functional Group Header
**Purpose:** Groups related transaction sets of the same type

**Key Elements:**
- GS01: Functional Identifier Code (HP=835, HC=837)
- GS02: Application Sender's Code
- GS03: Application Receiver's Code
- GS04: Date (CCYYMMDD)
- GS05: Time (HHMM or HHMMSS)
- GS06: Group Control Number (unique per group)
- GS08: Version/Release/Industry ID Code (005010X221A1, etc.)

**Example:**
```
GS*HP*SENDER*RECEIVER*20201231*1234*1*X*005010X221A1~
```

#### ST - Transaction Set Header
**Purpose:** Begins a specific transaction set

**Key Elements:**
- ST01: Transaction Set Identifier Code (835, 837, etc.)
- ST02: Transaction Set Control Number
- ST03: Implementation Convention Reference (005010X221A1)

**Example:**
```
ST*835*0001*005010X221A1~
```

#### SE - Transaction Set Trailer
**Purpose:** Ends transaction set and provides segment count

**Example:**
```
SE*24*0001~
```

#### GE - Functional Group Trailer
**Purpose:** Ends functional group and provides transaction set count

**Example:**
```
GE*1*1~
```

#### IEA - Interchange Control Trailer
**Purpose:** Ends interchange and provides functional group count

**Example:**
```
IEA*1*000000001~
```

---

## Loops and Segments

### Loop Concept

**Loop:** A repeating group of related segments identified by a loop ID (e.g., 2100, 2010BB, 2330A)

**Characteristics:**
- Hierarchical structure (loops can contain loops)
- Maximum repeat count defined by standard
- Identified by first segment in loop (e.g., CLP starts Loop 2100)

**Example Loop 2100 (835 Claim Payment Information):**
```
CLP*CLAIM123*1*1000.00*800.00*50.00*12*PATIENT123~  ← Loop starts
CAS*CO*45*50.00~                                      ← Adjustment info
NM1*QC*1*SMITH*JOHN~                                 ← Patient name
DTM*232*20201215~                                    ← Service date
                                                      ← Loop ends when next CLP or different loop starts
```

### Segment Concept

**Segment:** A single line of data consisting of a segment ID and data elements separated by delimiters

**Format:**
```
SEGMENT_ID*ELEMENT1*ELEMENT2*ELEMENT3~
```

**Parts:**
- **Segment Identifier:** 2-3 character code (CLP, NM1, REF, etc.)
- **Data Elements:** Values separated by element separator (usually *)
- **Segment Terminator:** Usually ~ (tilde)

**Example:**
```
NM1*PR*2*BLUE CROSS***PI*12345~
```
- Segment ID: NM1
- Elements: PR, 2, BLUE CROSS, (empty), (empty), (empty), PI, 12345

---

## Common Segment Types

### NM1 - Individual or Organizational Name

**Purpose:** Identifies parties (patients, providers, payers, etc.)

**Key Elements:**
- NM101: Entity Identifier Code (QC=Patient, PR=Payer, 82=Rendering Provider, etc.)
- NM102: Entity Type Qualifier (1=Person, 2=Non-Person)
- NM103: Name Last or Organization Name
- NM104: Name First
- NM105: Name Middle
- NM106: Name Prefix
- NM107: Name Suffix
- NM108: Identification Code Qualifier (MI, NPI, PI, XV, etc.)
- NM109: Identification Code
- NM110: Entity Relationship Code

**Example (Patient):**
```
NM1*QC*1*SMITH*JOHN*M***MI*123456789A~
```

**Example (Payer):**
```
NM1*PR*2*MEDICARE***PI*MEDICARE~
```

### REF - Reference Identification

**Purpose:** Provides additional reference numbers (claim numbers, control numbers, etc.)

**Key Elements:**
- REF01: Reference Identification Qualifier (codes vary by context)
- REF02: Reference Identification
- REF03: Description (optional)

**Example:**
```
REF*6R*CLAIM12345~
```

### DTM - Date/Time Reference

**Purpose:** Provides dates related to the transaction

**Key Elements:**
- DTM01: Date/Time Qualifier (codes define date type)
- DTM02: Date (CCYYMMDD or CCYYMM)
- DTM03: Time (HHMM or HHMMSS) - optional

**Example:**
```
DTM*232*20201231~
```

### AMT - Monetary Amount

**Purpose:** Specifies monetary values

**Key Elements:**
- AMT01: Amount Qualifier Code (defines amount type)
- AMT02: Monetary Amount
- AMT03: Credit/Debit Flag Code (optional)

**Example:**
```
AMT*D*250.00~
```

### QTY - Quantity

**Purpose:** Specifies quantity information

**Key Elements:**
- QTY01: Quantity Qualifier
- QTY02: Quantity
- QTY03: Composite Unit of Measure (optional)

**Example:**
```
QTY*CA*5~
```

---

## Entity Identifier Codes (NM101)

Common entity codes used across healthcare transactions:

| Code | Entity Type | Usage |
|------|-------------|-------|
| **QC** | Patient | Patient name in claims/payments |
| **IL** | Insured/Subscriber | Subscriber name when different from patient |
| **PR** | Payer | Insurance company/payer |
| **PE** | Payee | Payment recipient |
| **82** | Rendering Provider | Provider who rendered service |
| **77** | Service Location | Where service was performed |
| **74** | Corrected Insured | Corrected subscriber information |
| **TT** | Crossover Carrier | Coordination of benefits carrier |
| **DQ** | Other Subscriber | Other subscriber information |

**Note:** Entity codes are context-specific to transaction types. See transaction-specific skills for complete lists.

---

## Parsing Architecture Pattern

### Master + Split Table Pattern

**Concept:** Parser writes all occurrences of a segment type to a master table, then creates split tables filtered by specific criteria.

**Why:**
- **Flexibility:** Segment can appear in multiple loops with different meanings
- **Maintainability:** Single source of truth in master table
- **Query Performance:** Split tables optimized for specific use cases

**Example: NM1 Segments**

```
Master Table: NM1_MASTER
├─ Contains ALL NM1 segments from transaction
├─ Includes section_code field to identify loop/context
└─ Columns match X12 NM1 structure (NM101-NM110)

Split Tables:
├─ nm1_2100_patient (WHERE section_code ILIKE '%nm1_2100%')
├─ nm1_2010bb_payer (WHERE section_code ILIKE '%nm1_2010bb%')
├─ nm1_2010aa_billing_provider (WHERE section_code ILIKE '%nm1_2010aa%')
└─ ... (one split table per entity type/loop combination)
```

**SQL Pattern:**
```sql
-- Parser writes to master
INSERT INTO LAKE_MEDEDI.NM1_MASTER (...) VALUES (...);

-- Split table creation (via data flow)
CREATE TABLE STG_MEDEDI.nm1_2100_patient AS
SELECT * FROM LAKE_MEDEDI.NM1_MASTER
WHERE section_code ILIKE '%nm1_2100%';

CREATE TABLE STG_MEDEDI.nm1_2010bb_payer AS
SELECT * FROM LAKE_MEDEDI.NM1_MASTER
WHERE section_code ILIKE '%nm1_2010bb%';
```

**Critical Constraint:**
- Split tables MUST have identical column schemas as master
- Table names provide context, not column names
- Pattern relies on `SELECT *` for split insertion

---

## Key Design Principles

### 1. Section Code Pattern

Use hierarchical section codes to track loop/segment position:

```
section_code format: parent_loop|child_loop|segment_id|occurrence

Examples:
- "2100|nm1_2100|NM1|1" (First NM1 in Loop 2100)
- "2100|2110|svc|1" (Service line in claim)
- "2010bb|nm1|NM1|1" (Payer name in Loop 2010BB)
```

### 2. Parent-Child Relationships

Use hash keys to link related segments:

```
parent_key_hash: Hash of parent loop identifier
key_hash: Hash of current segment identifier
file_key: Links all segments to source file
```

### 3. Schema Naming Convention

```
LAKE_[TRANSACTION]: Raw parsed data (mirrors X12 structure)
STG_[TRANSACTION]: Staging layer (split tables, transformations)
EDW_[TRANSACTION]: Enterprise data warehouse (denormalized, analytics-ready)
SYS_CONTROL: System metadata (files, events, processing state)
```

### 4. Table Naming Convention

**Master Tables:**
- Segment type + "_MASTER" (e.g., NM1_MASTER, CLP_MASTER)
- Contains all occurrences regardless of loop/context

**Split Tables:**
- segment_loopid_entitydescription (e.g., nm1_2100_patient, clp_claim)
- Filtered view of master for specific use case
- Lowercase with underscores

**System Tables:**
- Descriptive names (files_catalog, events_log)
- Indicate purpose, not technical structure

---

## Detailed Reference Documentation

For implementation-specific details, consult the references directory:

### Parsing Challenges and Solutions
**File:** references/parsing-challenges.md

For troubleshooting and handling edge cases, consult references/parsing-challenges.md covering:
- Multiple occurrences of same segment in different contexts
- Hierarchical loop structure tracking
- Conditional segment handling (situational elements)
- Variable delimiter detection and configuration

### Metadata-Driven Configuration
**File:** references/metadata-architecture.md

For tbl_defs.yaml and input_map.yaml structure and usage, consult references/metadata-architecture.md covering:
- Table definition YAML structure (columns, data types, constraints)
- Split table creation configuration
- Benefits of metadata-driven approach

### Version Standards and Compatibility
**File:** references/version-compatibility.md

For HIPAA 5010 standards and version migration guidance, consult references/version-compatibility.md covering:
- Current version specifications (005010X221A1, etc.)
- Key changes from 4010 to 5010
- Version identification in GS08 segment

### Error Handling Strategies
**File:** references/error-handling.md

For error handling patterns and data quality management, consult references/error-handling.md covering:
- File-level error handling (ISA/GS/ST validation)
- Segment-level error recovery
- Data quality issue flagging

### Performance Optimization
**File:** references/performance-optimization.md

For bulk insert patterns and indexing strategies, consult references/performance-optimization.md covering:
- Bulk insert batching patterns
- Master and split table indexing strategies

### Industry Usage Patterns
**File:** references/industry-usage-patterns.md

For file size guidelines, memory management, and scaling projections, consult references/industry-usage-patterns.md covering:
- Typical file size ranges by scenario (small practice to clearinghouse batches)
- Memory management with bounded row buffering (10K row limit rationale)
- Batching vs streaming tradeoffs
- Scaling projections for production workloads

### Java Parser Implementation
**File:** references/java-parser-implementation.md

For BaseParser template, X12Segment class, ParseContext stack, RowAccumulator, qualifier-based lookup, and file key generation, consult references/java-parser-implementation.md covering:
- BaseParser template method pattern and transaction-specific implementations
- X12Segment immutable class with element positioning
- ParseContext push/pop pattern for hierarchy tracking
- RowAccumulator flush thresholds and memory-bounded batching
- Qualifier-based segment lookup for multi-purpose segments (NM1, REF, DTP, AMT)
- SHA-256 file key generation for parent-child linking

### Standards and Related Skills
**File:** references/standards-and-skills.md

For official documentation references and related transaction-specific skills, consult references/standards-and-skills.md covering:
- ASC X12, HIPAA, CMS official documentation
- Links to ali-x12-835-payment (835 Payment transaction specifics)
- Links to ali-x12-837i-institutional (837i Institutional Claims)

---

**Document Version:** 2.0 (Progressive Disclosure Refactor)
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Ingestion Engine Team

**Change Log:**
- v2.0 (2026-02-16): Progressive disclosure refactoring - moved 3,000 words to references/, added negative triggers for cluster coordination, created 8 reference files for on-demand loading
- v1.3 (2026-01-02): Added Java Parser Architecture sections (BaseParser template method, X12Segment class, ParseContext push/pop, RowAccumulator flush pattern, qualifier-based lookup, file key generation)
- v1.2 (2025-12-18): Added ISA Fixed-Length Requirement section with parser implementation guidance
- v1.1 (2025-12-17): Added Industry Usage Patterns section (file sizes, memory management, batching strategies)
