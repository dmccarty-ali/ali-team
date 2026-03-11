---
name: ali-hl7-v2-core
description: |
  HL7 v2.x messaging fundamentals for healthcare system integration. Use when:

  PLANNING: Designing ADT/lab/order integrations, interface engines, message routing

  IMPLEMENTATION: Parsing HL7 v2 messages, building segments, handling encoding rules

  GUIDANCE: Best practices for message types, acknowledgments, segment usage

  REVIEW: Validating HL7 v2 message structure, checking segment conformance, auditing interfaces
---

# HL7 v2.x Core Fundamentals

## Overview

HL7 Version 2.x is the workhorse of healthcare integration - a pipe-delimited messaging standard used for ADT (Admit/Discharge/Transfer), lab results, orders, scheduling, and more. Despite FHIR's rise, HL7 v2 remains dominant in hospital interfaces, with v2.3, v2.4, v2.5, and v2.5.1 most common in production.

---

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing HL7 v2 interfaces between systems
- Planning ADT, lab, or order integrations
- Architecting interface engine message routing
- Mapping HL7 v2 to/from other formats (FHIR, X12)

**Implementation:**
- Parsing HL7 v2 messages (pipe-delimited)
- Building or modifying HL7 v2 message segments
- Handling encoding characters and escape sequences
- Implementing ACK/NAK acknowledgment patterns

**Guidance/Best Practices:**
- Asking about HL7 v2 segment usage
- Which message type for a use case
- Acknowledgment best practices
- Z-segment conventions

**Review/Validation:**
- Checking HL7 v2 message conformance
- Validating segment structure and required fields
- Reviewing interface specifications
- Auditing message routing logic

---

## Key Principles

- **Pipe-delimited simplicity** - Messages are plain text with | (pipe), ^ (component), & (subcomponent) delimiters
- **Segment-based structure** - Each line is a segment identified by 3-character code (MSH, PID, OBR, etc.)
- **Trigger events drive messages** - Message type + event (ADT^A01, ORM^O01) determines content
- **MSH defines everything** - Encoding characters, sender/receiver, message type all in MSH
- **Acknowledgments are critical** - ACK/NAK pattern ensures reliable delivery
- **Z-segments for custom data** - Site-specific segments start with Z (ZPD, ZDG, etc.)
- **Versions are mostly additive** - v2.5 adds to v2.4, rarely breaks compatibility
- **Field positions are sacred** - PID-3 is always patient ID; positions don't change

---

## Message Structure

### Basic Format

```
MSH|^~\&|SENDING_APP|SENDING_FAC|RECEIVING_APP|RECEIVING_FAC|20240115120000||ADT^A01|MSG00001|P|2.5.1
EVN|A01|20240115120000
PID|1||12345678^^^HOSP^MR||Smith^John^Jacob||19700515|M|||123 Main St^^Anytown^CA^12345
PV1|1|I|ICU^101^A^HOSP||||1234567890^Jones^Sarah^M^^^MD|||MED|||||||1234567890^Jones^Sarah^M^^^MD|IP||||||||||||||||||||||||||20240115080000
```

### Structure Hierarchy

```
Message
├── Segment (one per line)
│   ├── Field (separated by |)
│   │   ├── Component (separated by ^)
│   │   │   └── Subcomponent (separated by &)
│   │   └── Repetition (separated by ~)
```

### Encoding Characters (MSH-2)

The five encoding characters are defined in MSH-2:

| Position | Character | Name | Purpose |
|----------|-----------|------|---------|
| 1 | `^` | Component separator | Separates components within a field |
| 2 | `~` | Repetition separator | Separates repeated occurrences |
| 3 | `\` | Escape character | Escapes special characters |
| 4 | `&` | Subcomponent separator | Separates subcomponents |
| 5 | `#` | Truncation character (v2.7+) | Indicates truncated data |

**Standard MSH-1 and MSH-2:**
```
MSH|^~\&|...
```
MSH-1 is the field separator (|), MSH-2 is ^~\&

---

## MSH - Message Header

The MSH segment defines the message envelope and MUST be the first segment.

```
MSH|^~\&|LABSYS|LAB|EHRSYS|HOSP|20240115143022||ORU^R01|MSG12345|P|2.5.1|||AL|NE
```

| Field | Name | Example | Description |
|-------|------|---------|-------------|
| MSH-1 | Field Separator | `\|` | Always pipe |
| MSH-2 | Encoding Characters | `^~\&` | Component, repetition, escape, subcomponent |
| MSH-3 | Sending Application | `LABSYS` | Source system identifier |
| MSH-4 | Sending Facility | `LAB` | Source facility |
| MSH-5 | Receiving Application | `EHRSYS` | Destination system |
| MSH-6 | Receiving Facility | `HOSP` | Destination facility |
| MSH-7 | Date/Time of Message | `20240115143022` | YYYYMMDDHHMMSS |
| MSH-8 | Security | | Usually empty |
| MSH-9 | Message Type | `ORU^R01` | Type^Event^Structure |
| MSH-10 | Message Control ID | `MSG12345` | Unique message identifier |
| MSH-11 | Processing ID | `P` | P=Production, D=Debug, T=Training |
| MSH-12 | Version ID | `2.5.1` | HL7 version |
| MSH-15 | Accept Ack Type | `AL` | AL=Always, NE=Never, ER=Error only |
| MSH-16 | Application Ack Type | `NE` | Same as above |

---

## Common Message Types

### ADT - Admit/Discharge/Transfer

Patient movement events.

| Event | Description | Key Use |
|-------|-------------|---------|
| A01 | Admit/Visit | Patient admitted to facility |
| A02 | Transfer | Patient transferred between units |
| A03 | Discharge | Patient discharged |
| A04 | Register | Outpatient registration |
| A05 | Pre-admit | Pre-admission created |
| A08 | Update | Patient information update |
| A11 | Cancel Admit | Cancel A01 |
| A13 | Cancel Discharge | Cancel A03 |
| A28 | Add Person | Add person to MPI (no visit) |
| A31 | Update Person | Update MPI person info |
| A34 | Merge | Merge patient records |
| A40 | Merge Patient | Merge patient identifier list |

**ADT^A01 Example:**
```
MSH|^~\&|ADT|HOSP|HIS|HOSP|20240115080000||ADT^A01|MSG001|P|2.5.1
EVN|A01|20240115080000
PID|1||12345678^^^HOSP^MR~987654321^^^SSA^SS||Smith^John^Jacob||19700515|M|||123 Main St^^Anytown^CA^12345||555-555-5555||M|CAT|12345678
PV1|1|I|ICU^101^A^HOSP||||1234567890^Jones^Sarah^^^MD|||MED|||||||1234567890^Jones^Sarah^^^MD|IP|VN12345|||||||||||||||||||||20240115080000
IN1|1|BCBS001|BCBS^Blue Cross Blue Shield|PO Box 12345^^City^ST^12345|||GRP12345||||20240101||||Smith^John|SELF|19700515|123 Main St^^Anytown^CA^12345
```

### ORM - Order Message

General orders (lab, radiology, pharmacy).

| Event | Description |
|-------|-------------|
| O01 | Order/Service Request |

**ORM^O01 Example:**
```
MSH|^~\&|CPOE|HOSP|LAB|LAB|20240115100000||ORM^O01|MSG002|P|2.5.1
PID|1||12345678^^^HOSP^MR||Smith^John||19700515|M
PV1|1|I|ICU^101^A
ORC|NW|ORD001|LAB001||SC|||1^Once^^^^R||20240115100000|NURSE001^Smith^Jane|||||||HOSP^General Hospital
OBR|1|ORD001|LAB001|80053^COMPREHENSIVE METABOLIC PANEL^CPT|||20240115100000|||||||||1234567890^Jones^Sarah^^^MD||||||20240115100000|||F
```

### ORU - Observation Result

Lab results, radiology reports, vital signs.

| Event | Description |
|-------|-------------|
| R01 | Unsolicited Observation Result |

**ORU^R01 Example:**
```
MSH|^~\&|LAB|LAB|HIS|HOSP|20240115140000||ORU^R01|MSG003|P|2.5.1
PID|1||12345678^^^HOSP^MR||Smith^John||19700515|M
PV1|1|I|ICU^101^A
ORC|RE|ORD001|LAB001||CM
OBR|1|ORD001|LAB001|80053^COMPREHENSIVE METABOLIC PANEL^CPT|||20240115100000|||||||||1234567890^Jones^Sarah^^^MD||||||20240115140000|||F
OBX|1|NM|2339-0^GLUCOSE^LN||95|mg/dL|70-100|N|||F|||20240115135500
OBX|2|NM|2823-3^POTASSIUM^LN||4.2|mmol/L|3.5-5.0|N|||F|||20240115135500
OBX|3|NM|2951-2^SODIUM^LN||140|mmol/L|136-145|N|||F|||20240115135500
```

### SIU - Scheduling

Appointment and scheduling events.

| Event | Description |
|-------|-------------|
| S12 | Notification of New Appointment |
| S13 | Notification of Appointment Reschedule |
| S14 | Notification of Appointment Modification |
| S15 | Notification of Appointment Cancellation |

### MDM - Medical Document Management

Clinical document notifications.

| Event | Description |
|-------|-------------|
| T01 | Original Document Notification |
| T02 | Original Document with Content |

---

## Key Segments

### PID - Patient Identification

```
PID|1||12345678^^^HOSP^MR~987654321^^^SSA^SS||Smith^John^Jacob||19700515|M|||123 Main St^^Anytown^CA^12345||555-555-5555||M|CAT|ACCT001
```

| Field | Name | Example | Notes |
|-------|------|---------|-------|
| PID-1 | Set ID | `1` | Sequence number |
| PID-2 | External ID | | (Deprecated in v2.3.1+) |
| PID-3 | Patient ID List | `12345678^^^HOSP^MR` | ID^Check^Code^Assigner^Type |
| PID-5 | Patient Name | `Smith^John^Jacob` | Family^Given^Middle^Suffix^Prefix |
| PID-7 | Date of Birth | `19700515` | YYYYMMDD |
| PID-8 | Sex | `M` | M, F, O, U, A, N |
| PID-11 | Address | `123 Main St^^Anytown^CA^12345` | Street^Other^City^State^Zip |
| PID-13 | Phone (Home) | `555-555-5555` | |
| PID-16 | Marital Status | `M` | S, M, D, W, etc. |
| PID-18 | Account Number | `ACCT001` | |

### PV1 - Patient Visit

```
PV1|1|I|ICU^101^A^HOSP||||1234567890^Jones^Sarah^^^MD|||MED|||||||1234567890^Jones^Sarah^^^MD|IP|VN12345|||||||||||||||||||||20240115080000
```

| Field | Name | Example | Notes |
|-------|------|---------|-------|
| PV1-1 | Set ID | `1` | |
| PV1-2 | Patient Class | `I` | I=Inpatient, O=Outpatient, E=Emergency |
| PV1-3 | Assigned Location | `ICU^101^A^HOSP` | Unit^Room^Bed^Facility |
| PV1-7 | Attending Doctor | `1234567890^Jones^Sarah^^^MD` | ID^Family^Given^Middle^Suffix^Prefix |
| PV1-10 | Hospital Service | `MED` | Medical, SUR, OBS, etc. |
| PV1-17 | Admitting Doctor | | Same format as PV1-7 |
| PV1-18 | Patient Type | `IP` | |
| PV1-19 | Visit Number | `VN12345` | |
| PV1-44 | Admit Date/Time | `20240115080000` | |

### ORC - Common Order

```
ORC|NW|ORD001|LAB001||SC|||1^Once^^^^R||20240115100000|NURSE001^Smith^Jane
```

| Field | Name | Example | Notes |
|-------|------|---------|-------|
| ORC-1 | Order Control | `NW` | NW=New, CA=Cancel, SC=Status Changed |
| ORC-2 | Placer Order Number | `ORD001` | Ordering system's ID |
| ORC-3 | Filler Order Number | `LAB001` | Performing system's ID |
| ORC-5 | Order Status | `SC` | SC=Scheduled, IP=In Progress, CM=Completed |
| ORC-7 | Quantity/Timing | | Frequency |
| ORC-9 | Date/Time of Transaction | `20240115100000` | |

### OBR - Observation Request

```
OBR|1|ORD001|LAB001|80053^COMPREHENSIVE METABOLIC PANEL^CPT|||20240115100000
```

| Field | Name | Example | Notes |
|-------|------|---------|-------|
| OBR-1 | Set ID | `1` | |
| OBR-2 | Placer Order Number | `ORD001` | |
| OBR-3 | Filler Order Number | `LAB001` | |
| OBR-4 | Universal Service ID | `80053^CMP^CPT` | Code^Text^System |
| OBR-7 | Observation Date/Time | `20240115100000` | Specimen collection time |
| OBR-16 | Ordering Provider | | |
| OBR-22 | Results Rpt/Status Change | | |
| OBR-25 | Result Status | `F` | F=Final, P=Preliminary, C=Corrected |

### OBX - Observation Result

```
OBX|1|NM|2339-0^GLUCOSE^LN||95|mg/dL|70-100|N|||F|||20240115135500
```

| Field | Name | Example | Notes |
|-------|------|---------|-------|
| OBX-1 | Set ID | `1` | |
| OBX-2 | Value Type | `NM` | NM=Numeric, ST=String, TX=Text, CE=Coded |
| OBX-3 | Observation Identifier | `2339-0^GLUCOSE^LN` | Code^Text^System (LOINC) |
| OBX-5 | Observation Value | `95` | Actual result |
| OBX-6 | Units | `mg/dL` | |
| OBX-7 | Reference Range | `70-100` | |
| OBX-8 | Abnormal Flags | `N` | N=Normal, L=Low, H=High, A=Abnormal |
| OBX-11 | Observation Result Status | `F` | F=Final, P=Preliminary |
| OBX-14 | Date/Time of Observation | `20240115135500` | |

### IN1 - Insurance

```
IN1|1|BCBS001|BCBS^Blue Cross Blue Shield|PO Box 12345^^City^ST^12345|||GRP12345||||20240101||||Smith^John|SELF|19700515
```

| Field | Name | Example | Notes |
|-------|------|---------|-------|
| IN1-1 | Set ID | `1` | Sequence (1=primary, 2=secondary) |
| IN1-2 | Insurance Plan ID | `BCBS001` | |
| IN1-3 | Insurance Company ID | `BCBS` | |
| IN1-4 | Insurance Company Name | `Blue Cross...` | |
| IN1-8 | Group Number | `GRP12345` | |
| IN1-12 | Plan Effective Date | `20240101` | |
| IN1-16 | Name of Insured | `Smith^John` | |
| IN1-17 | Relationship | `SELF` | SELF, SPO, CHD, etc. |

---

## Acknowledgments (ACK)

### ACK Structure

```
MSH|^~\&|RECEIVING_APP|RECEIVING_FAC|SENDING_APP|SENDING_FAC|20240115143025||ACK^R01|ACK12345|P|2.5.1
MSA|AA|MSG12345
```

### MSA - Message Acknowledgment

| Field | Name | Values |
|-------|------|--------|
| MSA-1 | Acknowledgment Code | AA=Accepted, AE=Error, AR=Rejected |
| MSA-2 | Message Control ID | Original message's MSH-10 |
| MSA-3 | Text Message | Error description if AE/AR |

### Acknowledgment Codes

| Code | Name | Meaning |
|------|------|---------|
| AA | Application Accept | Message accepted and processed |
| AE | Application Error | Message accepted but error in processing |
| AR | Application Reject | Message rejected - don't resend |
| CA | Commit Accept | Message committed to safe storage |
| CE | Commit Error | Commit failed |
| CR | Commit Reject | Message cannot be committed |

### ERR Segment (v2.5+)

```
MSH|^~\&|RECEIVING_APP|RECEIVING_FAC|SENDING_APP|SENDING_FAC|20240115143025||ACK^R01|ACK12345|P|2.5.1
MSA|AE|MSG12345|Invalid patient identifier
ERR||PID^1^3|103^Unknown identifier^HL70357|E|||Invalid MRN format
```

---

## Data Types

### XPN - Extended Person Name

```
Smith^John^Jacob^Jr^Dr^MD
```
Family^Given^Middle^Suffix^Prefix^Degree

### XAD - Extended Address

```
123 Main St^Apt 4B^Anytown^CA^12345^USA^M
```
Street^Other^City^State^Zip^Country^Type

### XTN - Extended Telephone

```
(555)555-5555^PRN^PH^email@example.com
```
Number^Use^Type^Email

**Use Codes:** PRN=Primary Residence, WPN=Work, ORN=Other Residence
**Type Codes:** PH=Phone, FX=Fax, CP=Cell, BP=Beeper

### CX - Extended Composite ID

```
12345678^^^HOSP^MR
```
ID^Check Digit^Code^Assigning Authority^Type Code

**Common Type Codes:**
- MR = Medical Record Number
- PI = Patient Internal Identifier
- SS = Social Security Number
- AN = Account Number

### CWE/CE - Coded With Exceptions / Coded Element

```
80053^COMPREHENSIVE METABOLIC PANEL^CPT
```
Code^Text^Coding System^Alternate Code^Alternate Text^Alternate System

---

## Escape Sequences

When data contains delimiter characters, use escape sequences:

| Sequence | Character | Meaning |
|----------|-----------|---------|
| `\F\` | `\|` | Field separator |
| `\S\` | `^` | Component separator |
| `\T\` | `&` | Subcomponent separator |
| `\R\` | `~` | Repetition separator |
| `\E\` | `\` | Escape character |
| `\.br\` | | Line break |
| `\X0D\` | | Hex character (carriage return) |

**Example:**
```
OBX|1|TX|NOTES||Patient reports pain level 8/10\T\worse in morning\.br\Prescribed ibuprofen.
```

---

## Parsing Patterns

### Basic Parser Logic

```
1. Read line-by-line (segment terminator is usually \r or \r\n)
2. First 3 characters = segment ID (MSH, PID, etc.)
3. For MSH: field separator is char 4 (|), encoding chars are field 2
4. Split remaining segments by field separator
5. Split fields by component separator (^)
6. Split components by subcomponent separator (&)
7. Handle repetitions (~)
8. Process escape sequences
```

### MSH Special Handling

MSH is unique - the field separator IS the first field:

```
MSH|^~\&|...
   ^
   |-- This | is MSH-1
      ^~~~
      |-- This is MSH-2 (encoding characters)
```

So MSH field numbering:
- MSH-1 = | (the separator itself)
- MSH-2 = ^~\& (encoding characters)
- MSH-3 = Sending Application (first field after MSH-2)

---

## Z-Segments

Custom segments for site-specific data. Always start with Z.

```
ZPD|1|EMPLOYEE|DEPT123|20200101||ALLERGIES_VERIFIED|20240115
```

**Conventions:**
- Document Z-segments in interface specification
- Use consistent naming across interfaces
- Avoid duplicating data already in standard segments
- Consider future migration to standard segments

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Hardcoding delimiters | Some systems use non-standard chars | Parse MSH-1 and MSH-2 dynamically |
| Ignoring escape sequences | Data corruption, parsing failures | Implement full escape handling |
| Assuming segment order | Order can vary by message type | Parse by segment ID, not position |
| Not handling repeating fields | Miss multiple IDs, phones, etc. | Split on ~ and process all |
| Treating empty as null | Empty string vs missing have meaning | Distinguish "" from absent field |
| Skipping ACK processing | Message loss, duplicate processing | Implement proper ACK/retry |
| Ignoring version differences | Fields added/moved between versions | Check MSH-12 and handle accordingly |
| Not validating required fields | Downstream failures | Validate before processing |

---

## Quick Reference

### OBX Value Types

| Code | Type | Example |
|------|------|---------|
| NM | Numeric | 95 |
| ST | String | Normal |
| TX | Text | Free text paragraph |
| CE/CWE | Coded Element | 73211009^Diabetes^SCT |
| DT | Date | 20240115 |
| TM | Time | 143022 |
| TS | Timestamp | 20240115143022 |
| FT | Formatted Text | With escape sequences |
| ED | Encapsulated Data | Base64 encoded |

### Common Abnormal Flags

| Flag | Meaning |
|------|---------|
| N | Normal |
| L | Low |
| H | High |
| LL | Critically Low |
| HH | Critically High |
| A | Abnormal |
| AA | Very Abnormal |

### Result Status Codes

| Code | Meaning |
|------|---------|
| F | Final |
| P | Preliminary |
| C | Corrected |
| X | Cancelled |
| I | Pending |

### Order Control Codes

| Code | Meaning |
|------|---------|
| NW | New Order |
| OK | Order Accepted |
| CA | Cancel |
| DC | Discontinue |
| RE | Observations |
| SC | Status Changed |
| XO | Change Order |

---

## HL7 v2 to FHIR Mapping

| HL7 v2 | FHIR Resource |
|--------|---------------|
| PID | Patient |
| PV1 | Encounter |
| ORC/OBR | ServiceRequest |
| OBX | Observation, DiagnosticReport |
| DG1 | Condition |
| IN1/IN2 | Coverage |
| GT1 | RelatedPerson (guarantor) |
| NK1 | RelatedPerson |
| AL1 | AllergyIntolerance |
| RXA | Immunization, MedicationAdministration |

---

## References

- [HL7 v2.5.1 Specification](https://www.hl7.org/implement/standards/product_brief.cfm?product_id=144)
- [HL7 v2 Message Structure](https://hl7-definition.caristix.com/v2/)
- [HL7 v2 to FHIR Mapping](https://build.fhir.org/ig/HL7/v2-to-fhir/)
- [HAPI HL7 v2 Library](https://hapifhir.github.io/hapi-hl7v2/)
