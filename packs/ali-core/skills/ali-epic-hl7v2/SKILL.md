---
name: ali-epic-hl7v2
description: |
  Epic-specific HL7 v2 interface patterns and Bridges engine configuration. Use when:

  PLANNING: Designing ADT/ORM/ORU interfaces into or out of Epic, planning Z-segment
  customizations, architecting real-time vs batch interface strategies with Epic Bridges

  IMPLEMENTATION: Configuring Epic Bridges interfaces, writing Epic-specific segment
  transformations, handling Epic ADT event codes, implementing Epic Interconnect middleware

  GUIDANCE: Asking about Epic's HL7 v2 deviations from standard, which message types Epic
  supports, how Epic handles Z-segments, Epic-specific field usage in MSH/PID/PV1

  REVIEW: Validating HL7 v2 interface specs against Epic's implementation, checking
  ADT event code coverage, auditing Z-segment definitions for Epic compatibility
---

# Epic HL7 v2 Interface Patterns

## Overview

Epic's HL7 v2 implementation runs through the **Bridges** interface engine. While Epic supports standard HL7 v2 message structures, it has specific requirements, Z-segment conventions, and field usages that differ from the baseline standard. This skill supplements `ali-hl7-v2-core` with the Epic-specific layer. Load both skills when working on Epic HL7 v2 integrations.

---

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing ADT, ORM, or ORU interfaces with Epic as the sender or receiver
- Planning Epic Bridges driver configuration for new interface connections
- Deciding between real-time and batch HL7 v2 processing for Epic
- Mapping third-party system HL7 v2 feeds to Epic's expected format
- Designing Epic Interconnect middleware deployments

**Implementation:**
- Configuring Epic Bridges interface drivers (HL7 receive/send)
- Writing Epic-specific segment transformations or Z-segment definitions
- Implementing acknowledgment (ACK/NAK) handling for Epic interfaces
- Building ADT notification feeds from Epic to downstream systems
- Implementing ORM order messages or ORU result messages with Epic

**Guidance/Best Practices:**
- Asking how Epic deviates from standard HL7 v2 segment definitions
- Asking which ADT event codes Epic generates or accepts
- Asking about Epic Z-segment naming conventions or required Z-segments
- Asking about Epic Interconnect vs direct Bridges connections

**Review/Validation:**
- Reviewing an HL7 v2 interface specification for Epic compatibility
- Checking ADT event code coverage against Epic's supported list
- Validating Z-segment definitions against Epic conventions
- Auditing MLLP connection settings for Epic Bridges

## When NOT to Use This Skill

- For base HL7 v2 message structure, encoding rules, or acknowledgment patterns not specific to Epic — use `ali-hl7-v2-core` instead
- For Epic FHIR R4 API integrations — use `ali-epic-fhir-r4`
- For SMART on FHIR, OAuth, or CDS Hooks — use `ali-epic-smart-auth`
- For Epic developer sandbox setup, rate limits, or general developer platform questions — use `ali-epic-integration-patterns`

---

## Key Principles

- Epic Bridges is the interface engine — every HL7 v2 connection goes through it; there is no way to send HL7 directly to Epic's application layer
- Epic's HL7 v2 implementation is version 2.3 / 2.4 / 2.5 depending on the message type and interface driver — never assume a specific version without checking the interface specification
- Z-segments are additive — Epic uses them to pass Epic-specific data not covered by standard segments; receiving systems must be able to accept and ignore unknown Z-segments
- Epic generates MSH-3 (Sending Application) and MSH-4 (Sending Facility) values from Bridges configuration — these must match what downstream systems expect
- PID-3 (Patient Identifier List) in Epic typically contains multiple repetitions — at minimum MRN and Enterprise MRN; integrations must handle the CX data type correctly
- Inbound interfaces require Epic-side mapping rules in Bridges to translate external identifiers to Epic internal IDs; never bypass this
- Real-time interfaces use MLLP over TCP; batch interfaces use file-based HL7 (often SFTP); the choice is driven by latency requirements, not convenience
- Epic Interconnect is the web services middleware layer — it exposes Epic Bridges routes over SOAP/REST wrappers for systems that cannot speak MLLP directly

---

## Epic Bridges Interface Engine

Epic Bridges is the HL7 interface engine embedded in Epic. It handles all inbound and outbound HL7 v2 message routing. Key concepts:

**Interface Drivers:** Each connection between Epic and an external system is configured as a driver. Drivers define connection type (MLLP, file), message types accepted, and transformation rules.

**Filter/Conversion Programs:** Epic uses its internal scripting to map incoming HL7 segments to Epic data fields and to build outgoing messages. These are configured in the Epic interface build, not as external code.

**Router Rules:** Messages arriving on a receive driver can be routed to multiple downstream Epic processes based on message content (event type, sending facility, etc.).

**Epic-generated Interface:** When Epic sends ADT or result messages to downstream systems, the message structure is generated by Bridges based on the driver configuration. The downstream system cannot change what Epic sends without modifying the Bridges configuration.

---

## Supported Message Types

### ADT (Admit/Discharge/Transfer)

Epic generates and accepts ADT messages as the primary patient movement notification mechanism.

**Outbound from Epic (Epic as sender):**

| Event | Trigger | Notes |
|-------|---------|-------|
| A01 | Admit patient | Includes full PID, PV1, DG1 |
| A02 | Transfer patient | PV1 updated with new location |
| A03 | Discharge patient | PV1-36 (discharge datetime) populated |
| A04 | Register outpatient | Common for ambulatory registration |
| A05 | Pre-admit | Used for scheduled inpatient pre-registration |
| A06 | Change outpatient to inpatient | Epic-specific workflow event |
| A07 | Change inpatient to outpatient | Epic-specific workflow event |
| A08 | Update patient information | Triggers on demographic changes; highest volume event |
| A11 | Cancel admit | Reversal of A01 |
| A12 | Cancel transfer | Reversal of A02 |
| A13 | Cancel discharge | Reversal of A03 |
| A31 | Update person information | Master Patient Index (MPI) updates; differs from A08 |
| A34 | Merge patient — permanent | Used after MPI merge in Epic |

**Inbound to Epic (Epic as receiver):**
- Epic accepts inbound ADT from external systems to pre-populate registration data
- A01/A04/A05 for pre-registration feeds (common from scheduling or pre-auth systems)
- A08 for demographic update feeds (common from MPI systems)
- Epic Bridges maps external patient identifiers to Epic internal IDs using PID-3

### ORM (Orders)

Epic generates ORM O01 messages for orders placed in Epic that need to route to external systems (lab, radiology, pharmacy).

- ORC segment carries the common order fields (order control, placer/filler IDs)
- OBR segment carries the observation/order detail
- Epic populates OBR-4 with the procedure code using the code set configured in Bridges (CPT, local, LOINC)
- NTE segment appended when clinical notes or order comments are present

### ORU (Observations/Results)

Epic accepts ORU R01 inbound from lab and radiology systems. Epic also generates ORU R01 outbound when results need to be sent to external systems.

- OBX segment carries individual result values
- Epic maps OBX-3 (observation identifier) to its internal test codes via translation tables configured in Bridges
- Numeric results: OBX-5 value with OBX-7 reference range and OBX-8 abnormal flag
- Coded results (e.g., pathology): OBX-2 = CE, OBX-5 = coded value

---

## Epic Z-Segment Conventions

Epic uses Z-segments to carry Epic-specific data not available in standard HL7 v2 segments.

**Naming convention:** Epic Z-segments are prefixed `ZEP` (Epic-authored) or named by the interface team per project. Common examples:

| Z-Segment | Purpose |
|-----------|---------|
| ZEP | Epic patient-specific data (enterprise MRN, primary care provider ID) |
| ZPD | Extended patient demographics not in PID |
| ZPV | Extended visit data not in PV1 |

**Key rules:**
- Z-segments always appear after all standard segments in a message group
- Receiving systems MUST accept and ignore unknown Z-segments — a system rejecting a message because of an unknown Z-segment is a receiving-side defect
- Epic does not use Z-segments to convey data that standard segments could carry — if PID or PV1 has a standard field for the data, Epic uses it
- Z-segment field definitions are documented in the Epic interface specification delivered to each customer; they are not public

---

## Epic-Specific Segment Notes

### MSH Segment

- MSH-3 (Sending Application): Set in Bridges driver config — must match downstream system expectation
- MSH-4 (Sending Facility): Set in Bridges driver config — typically the Epic instance name or organization
- MSH-9 (Message Type): Epic uses standard values (ADT^A01, ORU^R01, ORM^O01)
- MSH-11 (Processing ID): `P` for production, `T` for test — always verify this on new interfaces
- MSH-12 (Version ID): Typically 2.3 or 2.4 for ADT; 2.5 for ORU/ORM on newer Epic builds

### PID Segment

- PID-3 (Patient Identifier List): Multiple repetitions; includes MRN (typed as MR), Enterprise MRN (typed as EPI), and optionally SSN-derived ID
- PID-5 (Patient Name): XPN data type; Epic sends family^given^middle initial
- PID-8 (Sex): Epic uses M/F/U/O per HL7; not all downstream systems handle U or O
- PID-11 (Patient Address): Multiple repetitions possible (home, mailing, work)
- PID-18 (Patient Account Number): Financial account number, not the MRN — a common point of confusion

### PV1 Segment

- PV1-2 (Patient Class): I (inpatient), O (outpatient), E (emergency), P (preadmit) — standard values
- PV1-3 (Assigned Patient Location): PL data type — point of care ^ room ^ bed ^ facility
- PV1-7 (Attending Doctor): XCN data type; Epic sends NPI in the identifier component
- PV1-17 (Admitting Doctor): Separate from attending; populated on inpatient admissions
- PV1-44 (Admit Date/Time): TS format — verify timezone handling; Epic stores in local time

### OBR Segment

- OBR-2 (Placer Order Number): Epic's internal order ID (EAP ID or order ID)
- OBR-3 (Filler Order Number): External system's order number when Epic is the sender
- OBR-4 (Universal Service Identifier): Procedure code — code set (CPT vs LOINC vs local) varies by Bridges configuration
- OBR-7 (Observation Date/Time): Collection time for labs, procedure time for radiology
- OBR-25 (Result Status): F (final), P (preliminary), C (correction) — downstream systems must handle all three

---

## Real-Time vs Batch Processing

### Real-Time (MLLP)

- TCP socket connection, persistent or on-demand
- MLLP framing: `0x0B` start block, `0x1C 0x0D` end block
- Epic Bridges acts as both MLLP client (outbound) and MLLP server (inbound)
- Use when: downstream system requires immediate processing, latency under 30 seconds required
- ACK handling: Epic expects ACK AA (accept), ACK AE (error), ACK AR (reject); AE/AR trigger retry logic

### Batch (File-Based)

- HL7 batch protocol: FHS/BHS envelope around multiple messages
- Delivered via SFTP or shared filesystem
- Epic can generate scheduled batch extracts (nightly ADT census, result batch)
- Use when: high volume (100K+ messages/day), latency tolerance > 1 hour, receiving system cannot sustain persistent connections
- ACK: Epic batch does not expect real-time ACK; error reporting is separate

---

## Common Integration Scenarios

### ADT Feed to Downstream System

Epic generates ADT messages in near-real-time as patient events occur. The downstream system (HIE, scheduling system, billing, third-party analytics) receives the feed via MLLP.

**Configuration checklist:**
- Outbound Bridges driver configured with correct IP/port of downstream system
- Message filter rules defined (which event types to send, which facilities)
- Retry logic configured (Epic retries on failed ACK; downstream must handle duplicate messages)
- Downstream system must handle A08 volume — on large Epic instances, A08 can be 10x the volume of A01

### Inbound Lab Results (ORU R01)

Lab system sends ORU R01 to Epic via inbound Bridges driver. Epic maps the result to the correct order and patient.

**Key mapping requirements:**
- OBR-2 or OBR-3 must contain a value Epic can use to identify the originating order
- OBX-3 observation identifiers must be in Epic's translation tables (LOINC codes or custom local codes configured in Bridges)
- OBX-14 (date/time of observation) must be populated — Epic rejects results without it
- Results without a matching order require a Bridges "create order on result" configuration

### Inbound Order Feed (ORM O01)

External system sends orders to Epic. Less common — Epic is typically the order-originating system. Use case: remote scheduling system or order routing clearinghouse.

**Epic-specific requirement:** ORC-12 (ordering provider) must contain a valid NPI that Epic can resolve to a provider in its system. Orders with unresolvable providers will fail in Bridges unless a default provider mapping is configured.

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Assuming standard HL7 v2 fields work without checking Epic's implementation | Epic populates or requires specific fields differently — PID-3 with multiple repetitions, OBR-4 code set configuration, PV1-7 with NPI | Read the Epic interface specification for each interface; do not assume standard field usage |
| Sending a single PID-3 repetition with just an MRN | Epic frequently carries multiple identifiers in PID-3; inbound processing logic that overwrites or ignores extra repetitions will break Epic's patient matching | Handle PID-3 as a repeating field; preserve all identifier types |
| Treating A08 as a low-volume event | A08 fires on any demographic or visit update; on large hospitals it is the highest-volume event — receivers must be designed for it | Build idempotent A08 handlers; consider filtering A08 by specific PID or PV1 field changes if downstream cannot handle the volume |
| Hardcoding MSH-3 and MSH-4 values in receiving systems | Epic sends what Bridges is configured to send; if the Bridges config changes, the receiving system breaks | Make sending application and sending facility configurable; validate against an allowlist, not a hardcoded string |
| Building logic that requires real-time ACK for every batch message | Batch HL7 uses FHS/BHS envelope and does not produce per-message real-time ACKs | Use MLLP for real-time ACK-required workflows; reserve file-based batch for high-volume, latency-tolerant scenarios |
| Rejecting messages with unknown Z-segments | Unknown Z-segments are an extension mechanism, not an error | Configure receiving parsers to accept and discard unknown segments |
| Using the first OBX value from a multi-OBX result message | Epic sends panel results as multiple OBX segments under a single OBR | Iterate all OBX groups; associate each with OBX-1 (set ID) and OBX-3 (observation identifier) |

---

## Quick Reference

### ADT Event Codes Epic Generates (Common)

```
A01 - Admit / Visit Notification
A02 - Transfer
A03 - Discharge
A04 - Register Outpatient
A05 - Pre-Admit
A06 - Change Outpatient to Inpatient
A07 - Change Inpatient to Outpatient
A08 - Update Patient Information (highest volume)
A11 - Cancel Admit
A12 - Cancel Transfer
A13 - Cancel Discharge
A31 - Update Person (MPI-level update)
A34 - Merge Patient (post-MPI merge)
```

### PID-3 Identifier Types (Epic Standard)

```
MR  - Medical Record Number (MRN)
EPI - Enterprise Patient Index (Enterprise MRN across facilities)
SS  - Social Security Number (if configured to send)
```

### OBX Result Status Values

```
F - Final result
P - Preliminary result (expect correction)
C - Corrected result (replaces prior result)
X - Results cannot be obtained
```

### MLLP Framing

```
Start Block:  0x0B (vertical tab)
End Block:    0x1C (file separator) + 0x0D (carriage return)
```

---

## References

- [Epic HL7 v2 Interface Documentation (requires Epic UserWeb access)](https://userweb.epic.com)
- [HL7 v2.x Official Specification](https://www.hl7.org/implement/standards/product_brief.cfm?product_id=185)
- [ali-hl7-v2-core skill](../ali-hl7-v2-core/SKILL.md) — base HL7 v2 fundamentals
