---
epic_version: "November 2025"
fhir_version: "R4 (4.0.1)"
last_reviewed: "2026-03-20"
sources:
  - url: "https://open.epic.com/Interface/FHIR"
    reliability: 0.95
    scraped: "2026-03-20"
  - url: "https://www.healthit.gov/isa/united-states-core-data-interoperability-uscdi"
    reliability: 0.95
    scraped: "2026-03-20"
---

# USCDI v3 Data Class to Epic FHIR Resource: Detailed Reference

This file provides the expanded reference for USCDI v3 data class to Epic FHIR R4 resource mapping including required search parameters and known constraints. Load on demand when implementing USCDI v3 compliance for an Epic integration.

---

## USCDI v3 Data Classes — Detailed Mapping

### Allergies and Intolerances

**FHIR Resource:** `AllergyIntolerance`

**Required search parameters:**
- `patient` — returns all allergies for the patient
- `clinical-status` — active | inactive | resolved

**Epic notes:**
- Epic returns both drug allergies and environmental allergies under AllergyIntolerance
- `AllergyIntolerance.reaction.manifestation` is coded using SNOMED CT
- No-known-allergy (NKA) is represented as a specific coded entry, not an empty Bundle

---

### Care Team Members

**FHIR Resource:** `CareTeam`

**Required search parameters:**
- `patient` — returns the care team for the patient
- `status` — active | proposed | suspended | inactive | entered-in-error

**Epic notes:**
- Epic returns the primary care provider and the treating team
- `CareTeam.participant.role` is coded using SNOMED CT role codes
- May include both internal Epic providers and external providers if documented

---

### Clinical Notes

**FHIR Resource:** `DocumentReference`

**Required search parameters:**
- `patient`
- `type` — LOINC code for note type (see table below)
- `date` — creation/update date range

**Common note type LOINC codes in Epic:**

| Note Type | LOINC Code |
|-----------|------------|
| Discharge Summary | 18842-5 |
| History and Physical | 34117-2 |
| Progress Note | 11506-3 |
| Operative Note | 11504-8 |
| Consultation Note | 11488-4 |
| Procedure Note | 28570-0 |
| Radiology Report | 18748-4 |
| Pathology Report | 11526-1 |
| Referral Note | 57133-1 |

**Epic notes:**
- `DocumentReference.content.attachment.url` provides a URL to retrieve the note content
- Note content is returned as HTML or plain text depending on Epic configuration
- Provider notes may be restricted from the patient-facing FHIR view (MyChart scope)

---

### Diagnostic Imaging

**FHIR Resources:** `ImagingStudy`, `DiagnosticReport` (with category: LAB or RAD)

**Required search parameters for ImagingStudy:**
- `patient`
- `started` — date range for study start

**Required search parameters for DiagnosticReport (radiology):**
- `patient`
- `category` — `http://loinc.org|LP29684-5` (radiology)
- `date`

**Epic notes:**
- Epic returns radiology reports as DiagnosticReport resources with `presentedForm` attachment
- ImagingStudy references the DICOM study; actual image retrieval requires DICOMweb or VNA, not the FHIR API
- Some Epic instances expose DICOM UIDs in `ImagingStudy.identifier` for cross-referencing with PACS

---

### Diagnostic Tests (Non-Lab, Non-Imaging)

**FHIR Resources:** `DiagnosticReport`, `Observation`

**Epic notes:**
- Pulmonary function tests, EKGs, and other diagnostic procedures are returned as DiagnosticReport
- Results are nested as `DiagnosticReport.result` references to Observation resources
- Use `_include=DiagnosticReport:result` to fetch Observations in the same request

---

### Goals

**FHIR Resource:** `Goal`

**Required search parameters:**
- `patient`
- `lifecycle-status` — active | completed | cancelled

**Epic notes:**
- Goals created in Epic care plans are returned as Goal resources
- `Goal.target` contains measurable targets; may be empty if only a qualitative goal is documented

---

### Health Concerns

**FHIR Resource:** `Condition`

**Filter:** `category` = `http://hl7.org/fhir/us/core/CodeSystem/condition-category|health-concern`

**Epic notes:**
- Separate from problem list items — use category to distinguish
- Health concerns may include social determinants of health (SDOH) entries if configured in Epic

---

### Immunizations

**FHIR Resource:** `Immunization`

**Required search parameters:**
- `patient`
- `date`
- `status` — completed | entered-in-error | not-done

**Epic notes:**
- CVX codes used in `Immunization.vaccineCode`
- `Immunization.occurrenceDateTime` is the administration date
- Historical immunizations (patient-reported) are included; `Immunization.primarySource` = false distinguishes them

---

### Laboratory

**FHIR Resources:** `Observation` (category: laboratory), `DiagnosticReport`

**Required search parameters for Observation:**
- `patient`
- `category` — `http://terminology.hl7.org/CodeSystem/observation-category|laboratory`
- `date`
- `code` — LOINC code for specific test

**Epic notes:**
- Lab panel results: parent DiagnosticReport with child Observation resources per analyte
- Reference ranges in `Observation.referenceRange`
- Abnormal flags in `Observation.interpretation` (H, L, HH, LL, A)
- Pending results: `Observation.status` = registered or preliminary

---

### Medications

**FHIR Resources:** `MedicationRequest`, `MedicationDispense`, `Medication`

**Required search parameters for MedicationRequest:**
- `patient`
- `status` — active | completed | stopped | cancelled
- `intent` — order | plan

**Epic notes:**
- Active medication list: `MedicationRequest?patient={id}&status=active`
- `MedicationRequest.medication[x]` may be a CodeableConcept (RxNorm) or a reference to a `Medication` resource — handle both
- Dispense records require `MedicationDispense` — separate query from MedicationRequest
- Medication reconciliation entries may appear as `MedicationStatement` on some Epic versions (check CapabilityStatement)

---

### Patient Demographics

**FHIR Resource:** `Patient`

**Epic USCDI v3 demographic elements:**

| USCDI v3 Element | FHIR Path | Notes |
|------------------|-----------|-------|
| First Name | `Patient.name.given[0]` | |
| Last Name | `Patient.name.family` | |
| Date of Birth | `Patient.birthDate` | |
| Sex (for clinical use) | `Patient.extension[sex-for-clinical-use]` | US Core extension |
| Gender Identity | `Patient.extension[individual-genderIdentity]` | US Core extension |
| Pronouns | `Patient.extension[individual-pronouns]` | US Core extension |
| Race | `Patient.extension[us-core-race]` | OMB categories |
| Ethnicity | `Patient.extension[us-core-ethnicity]` | OMB categories |
| Preferred Language | `Patient.communication.language` | |
| Current Address | `Patient.address` | |
| Phone | `Patient.telecom` (system: phone) | |
| Email | `Patient.telecom` (system: email) | |

---

### Problems (Problem List)

**FHIR Resource:** `Condition`

**Filter:** `category` = `http://terminology.hl7.org/CodeSystem/condition-category|problem-list-item`

**Required search parameters:**
- `patient`
- `clinical-status` — active | inactive | resolved

**Epic notes:**
- ICD-10-CM codes in `Condition.code.coding` (system: `http://hl7.org/fhir/sid/icd-10-cm`)
- SNOMED CT codes may also be present as additional codings
- Onset date in `Condition.onsetDateTime` or `Condition.onsetAge`

---

### Procedures

**FHIR Resource:** `Procedure`

**Required search parameters:**
- `patient`
- `date`
- `status` — completed | in-progress | not-done

**Epic notes:**
- CPT codes in `Procedure.code.coding` (system: `http://www.ama-assn.org/go/cpt`)
- SNOMED CT procedure codes may also be present
- `Procedure.performedDateTime` or `Procedure.performedPeriod`
- Surgical procedures and minor procedures are both returned

---

### Sexual Orientation and Gender Identity

**FHIR Resource:** `Observation` with specific LOINC codes

| Concept | LOINC Code |
|---------|-----------|
| Sexual Orientation | 76690-7 |
| Gender Identity | 76691-5 |

**Epic notes:**
- Returned as `Observation` resources with the LOINC code in `Observation.code`
- `Observation.valueCodeableConcept` uses SNOMED CT or local codes per the US Core profile
- Not all Epic instances are configured to collect and expose SOGI data — empty result sets are expected if not configured

---

### Smoking Status

**FHIR Resource:** `Observation`

**LOINC code:** 72166-2 (Tobacco smoking status NHIS)

**Epic notes:**
- `Observation.valueCodeableConcept` uses SNOMED CT smoking status codes
- Most recent observation represents current status; historical entries also available by date range

---

### Vital Signs

**FHIR Resource:** `Observation` (category: vital-signs)

**Required search parameters:**
- `patient`
- `category` — `http://terminology.hl7.org/CodeSystem/observation-category|vital-signs`
- `code` — LOINC code for specific vital
- `date`

**Common vital sign LOINC codes:**

| Vital Sign | LOINC Code |
|------------|-----------|
| Body Weight | 29463-7 |
| Body Height | 8302-2 |
| BMI | 39156-5 |
| Heart Rate | 8867-4 |
| Respiratory Rate | 9279-1 |
| Blood Pressure Systolic | 8480-6 |
| Blood Pressure Diastolic | 8462-4 |
| Body Temperature | 8310-5 |
| Oxygen Saturation | 59408-5 |
| Head Circumference | 9843-4 |

---

## Notes on Coverage Gaps

The following USCDI v3 data classes have variable or partial Epic FHIR support — verify with the specific Epic customer's environment:

- **Unique Device Identifiers (UDI):** Available via `Device` resource; coverage depends on whether the Epic instance tracks implantable devices
- **Provenance:** `Provenance` resource available on some Epic versions; verify CapabilityStatement support
- **Social Determinants of Health (SDOH):** SDOH Observations and Conditions are available if the Epic instance has SDOH screening configured

---

*Last reviewed: 2026-03-20. Verify against current open.epic.com documentation for the target Epic named release.*
