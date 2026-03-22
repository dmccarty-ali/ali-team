---
name: ali-epic-integration-patterns
description: |
  Epic developer platform operational knowledge: sandbox setup, rate limits, USCDI
  compliance, bulk FHIR export, release cadence, and common gotchas. Use when:

  PLANNING: Setting up Epic sandbox environments, planning USCDI v3 compliance mapping,
  designing bulk FHIR export pipelines, evaluating Epic's release cadence impact on
  integration timelines, planning Care Everywhere interoperability scenarios

  IMPLEMENTATION: Configuring Epic sandbox credentials, implementing bulk FHIR
  $export operations, handling Epic rate limit responses, working with MyChart
  patient-facing APIs, implementing TEFCA/QHIN exchange patterns

  GUIDANCE: Asking about Epic developer platform gotchas, Epic named release schedule,
  which Epic FHIR resources satisfy USCDI v3, how Epic bulk export compares to
  standard FHIR bulk spec, Epic Connection Hub submission process

  REVIEW: Auditing Epic integration implementations for rate limit compliance,
  checking USCDI v3 coverage, validating bulk export error handling, reviewing
  Care Everywhere configuration requirements
---

# Epic Integration Platform Patterns

## Overview

This skill covers the operational and platform-level knowledge for building Epic integrations beyond the individual API or message type level. It addresses the developer experience: sandbox setup, rate limits, USCDI compliance, the Epic release cycle, bulk data, patient-facing APIs, and the common pitfalls that surface in the Epic developer community. Load alongside `ali-epic-fhir-r4`, `ali-epic-smart-auth`, or `ali-epic-hl7v2` based on the integration type.

---

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Planning an Epic integration project and need to understand the sandbox workflow
- Evaluating how Epic's named release cadence affects an integration delivery timeline
- Determining which FHIR resources satisfy USCDI v3 requirements via Epic
- Planning a bulk FHIR export pipeline from Epic for analytics or population health
- Designing a Care Everywhere query for external patient data
- Evaluating TEFCA or QHIN participation requirements that involve Epic

**Implementation:**
- Setting up Epic FHIR sandbox access and test credentials
- Implementing $export and polling the content-location URL for bulk data
- Handling Epic rate limit 429 responses with backoff logic
- Working with MyChart patient-facing APIs for consumer-facing app features
- Implementing QHIN record location or exchange workflows

**Guidance/Best Practices:**
- Asking about common gotchas in Epic integrations from the developer community
- Asking how Epic's bulk export compares to the FHIR Bulk Data specification
- Asking about Epic Connection Hub (App Orchard) submission and certification requirements
- Asking what the current Epic named release is and what changed
- Asking about Epic's approach to USCDI and CommonWell/Carequality

**Review/Validation:**
- Auditing an Epic integration for rate limit handling compliance
- Checking USCDI v3 data class coverage in a FHIR integration design
- Validating bulk export error handling and retry logic
- Reviewing Care Everywhere or QHIN implementation approach

## When NOT to Use This Skill

- For Epic FHIR R4 resource-specific questions (Patient, Observation, MedicationRequest) — use `ali-epic-fhir-r4`
- For SMART on FHIR authorization, OAuth flows, or CDS Hooks — use `ali-epic-smart-auth`
- For Epic HL7 v2 interface and Bridges configuration — use `ali-epic-hl7v2`
- For non-Epic healthcare interoperability (Cerner, Meditech, general FHIR) — use `ali-fhir-r4-core` or `ali-hl7-v2-core`

---

## Key Principles

- Epic releases named versions twice per year (May and November); each named release may change API behavior, add new FHIR resources, or deprecate parameters — build version awareness into integration roadmaps
- Epic rate limits are real and enforced — never assume unlimited API throughput; always implement exponential backoff and honor 429 responses
- The Epic FHIR sandbox (`fhir.epic.com`) uses synthetic patients and is publicly accessible; production requires customer-specific credentials — never confuse the two
- USCDI compliance is Epic's baseline for what FHIR resources must be available; apps targeting USCDI v3 can use Epic as a compliant data source, but must map the correct FHIR resource and profile
- Bulk FHIR export from Epic follows the FHIR Bulk Data specification (Async pattern with polling) but has Epic-specific behaviors — particularly around supported resource types, parameter constraints, and export file formats
- Care Everywhere is Epic's patient data exchange network; it uses Carequality and CommonWell as the underlying frameworks — integration design must account for which network(s) are configured at the Epic instance
- Community and blog sources on Epic integration are often version-specific; always verify against the current open.epic.com documentation before implementing based on community guidance

---

## Epic Developer Sandbox Setup

### Epic FHIR Sandbox (fhir.epic.com)

Epic provides a public sandbox at `https://fhir.epic.com` for testing FHIR R4 API integrations without customer access.

**What it provides:**
- FHIR R4 endpoints for the full set of Epic's publicly documented resources
- Synthetic test patients with realistic clinical data (documented at fhir.epic.com)
- SMART on FHIR authorization flows using sandbox credentials
- Public test client ID (`client_id`) for SMART EHR launch testing

**Accessing the sandbox:**
1. Register at `fhir.epic.com` — account creation is self-service
2. Create an app in the sandbox environment to receive a `client_id`
3. For backend service testing: upload a public key to receive a non-secret backend service client
4. Use the documented test patient IDs for FHIR resource reads

**Sandbox limitations:**
- Synthetic data — not representative of production data volume or variety
- Not all Epic modules are exercised — some resource types return empty results even when the endpoint exists
- Sandbox does not reflect customer-specific Epic build configuration — a FHIR call that works in the sandbox may behave differently on a specific customer's Epic instance due to local configuration

### Customer-Specific Sandbox

For integrations targeting a specific Epic customer, the customer provides a non-production Epic environment (typically called "nonprod" or "stage"). This environment:
- Uses real Epic build configuration (vs. the public sandbox's generic build)
- Requires the customer to provision test credentials and authorize the app
- More representative of production behavior but still uses synthetic test patients
- Requires customer IT involvement to set up; plan for 2-4 week lead time

### App Registration Flow

```
Developer self-registers at fhir.epic.com (sandbox testing)
    → Upload public key (backend services) or configure redirect URIs (SMART apps)
    → Receive sandbox client_id
    → Test in public sandbox

Production deployment:
    → Submit app to Epic Connection Hub / App Orchard for review
    → Epic customer provisions the app in their Epic instance
    → Customer-specific client_id issued per Epic instance
    → Credentials are per-customer, per-environment (not global)
```

---

## Rate Limits and Throttling

Epic enforces rate limits on FHIR API calls. Limits are not uniformly published but the following behaviors are documented by the developer community and confirmed in Epic's API documentation.

### Known Rate Limit Behaviors

- Epic returns HTTP 429 (Too Many Requests) when rate limits are exceeded
- The `Retry-After` response header is included with 429 responses indicating when to retry (in seconds)
- Rate limits are applied per client_id (not per IP) — distributed services sharing a client_id share the rate limit budget
- Bulk export operations are subject to separate, more generous limits — they are designed for high-volume export and are throttled differently than individual resource reads

### Handling 429 Responses

```python
import time
import httpx
from typing import Optional

def epic_api_call_with_retry(
    client: httpx.Client,
    url: str,
    headers: dict,
    max_retries: int = 5,
    base_delay_seconds: float = 1.0,
) -> httpx.Response:
    for attempt in range(max_retries):
        response = client.get(url, headers=headers)

        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            if retry_after:
                delay = float(retry_after)
            else:
                delay = base_delay_seconds * (2 ** attempt)
            time.sleep(delay)
            continue

        return response

    raise EpicRateLimitExceeded(f"Rate limit exceeded after {max_retries} retries")
```

**Recommendations:**
- Always implement retry with backoff — never retry immediately on 429
- Honor the `Retry-After` header value if present; do not assume a fixed backoff
- Reduce request concurrency if encountering frequent 429 responses — high-concurrency clients exhaust rate budgets faster
- For bulk operations, prefer bulk FHIR export over looping individual resource reads — bulk export is designed for high-volume access

---

## USCDI v3 Compliance Mapping

USCDI (United States Core Data for Interoperability) v3 defines the minimum data set that certified health IT systems must support. Epic's FHIR R4 implementation maps to USCDI v3 data classes.

### USCDI v3 Data Classes and Epic FHIR Resources

| USCDI v3 Data Class | Epic FHIR Resource(s) |
|---------------------|----------------------|
| Patient Demographics | `Patient` |
| Allergies and Intolerances | `AllergyIntolerance` |
| Care Team Members | `CareTeam` |
| Clinical Notes | `DocumentReference` |
| Diagnostic Imaging | `ImagingStudy`, `DiagnosticReport` |
| Diagnostic Tests | `DiagnosticReport`, `Observation` |
| Encounter Information | `Encounter` |
| Goals | `Goal` |
| Health Concerns | `Condition` (category: health-concern) |
| Immunizations | `Immunization` |
| Laboratory | `Observation` (category: laboratory), `DiagnosticReport` |
| Medications | `MedicationRequest`, `MedicationDispense`, `Medication` |
| Patient Documents | `DocumentReference` |
| Pediatric Vital Signs | `Observation` (specific LOINC codes) |
| Problems | `Condition` (category: problem-list-item) |
| Procedures | `Procedure` |
| Provenance | `Provenance` |
| Reason for Referral | `ServiceRequest` |
| Sexual Orientation and Gender Identity | `Observation` (specific LOINC codes) |
| Smoking Status | `Observation` (LOINC 72166-2) |
| Unique Device Identifiers | `Device` |
| Vital Signs | `Observation` (category: vital-signs) |

**Important caveats:**
- Epic's support for specific USCDI v3 data classes may be gated by Epic version and module configuration
- Not every Epic customer has all modules configured — verify with the specific customer's Epic environment
- Some USCDI v3 data classes have multiple FHIR resource representations in Epic depending on clinical context

---

## Bulk FHIR Export

Epic supports FHIR Bulk Data Access (FHIR $export) for population-level data extraction. This is the correct mechanism for analytics, population health, and large-scale data pipelines — not looping through individual resource reads.

### Export Pattern (Async Polling)

**Step 1: Initiate export**
```
GET [base]/Patient/$export?_type=Patient,Observation,Condition
Authorization: Bearer {access_token}
Accept: application/fhir+json
Prefer: respond-async
```

Epic responds with:
```
HTTP 202 Accepted
Content-Location: https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/$export-poll-status?jobId=abc123
```

**Step 2: Poll for completion**
```
GET https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/$export-poll-status?jobId=abc123
Authorization: Bearer {access_token}
```

Responses:
- `202 Accepted` with `X-Progress` header — still processing
- `200 OK` — export complete; body contains manifest with download URLs
- `500` — export failed; check error detail

**Step 3: Download export files**
```
GET {download_url_from_manifest}
Authorization: Bearer {access_token}
```

Files are in NDJSON format (one FHIR resource JSON object per line).

### Epic Bulk Export Specifics

- **Supported resource types:** Not all FHIR resource types support bulk export in Epic — check open.epic.com for the current supported list
- **Patient compartment exports:** `Patient/$export` returns all resources in the patient compartment; system-level `$export` is available for backend service clients with appropriate system scopes
- **_since parameter:** Epic supports the `_since` parameter for incremental exports (resources updated after a given timestamp); use this for scheduled incremental pipelines
- **File size:** Epic export files are split at configurable size limits; the manifest may contain multiple files per resource type
- **Token expiry during long exports:** For large exports taking hours, the access token may expire during polling or download — implement token refresh between steps
- **Group-level export:** `Group/{id}/$export` allows export scoped to a specific patient cohort; group IDs are Epic-specific and must be known in advance

---

## MyChart Patient-Facing APIs

MyChart is Epic's patient portal. Epic exposes FHIR APIs that patient-facing apps can use to access patient data through MyChart.

**Key characteristics:**
- Uses SMART on FHIR standalone launch from the patient's perspective
- The patient authenticates to MyChart; the app receives patient-scoped tokens
- Data scope is limited to what the patient can see in MyChart — not the full clinical record
- Supports the FHIR $everything operation for patient summary download (used by patient apps)

**Common MyChart patient API use cases:**
- Patient-facing apps that display medications, lab results, appointments
- Consumer health apps integrating via Apple Health or Google Health Connect (uses FHIR APIs behind the scenes)
- Patient engagement platforms pulling appointment and care plan data

**MyChart limitations:**
- Some clinical data is restricted from the patient view (provider notes, certain diagnoses) — FHIR calls return only what the patient is authorized to see
- Patient tokens are scoped to the authenticated patient only — no cross-patient access
- Epic configures what data is surfaced in MyChart per customer; the same FHIR call may return different data at different Epic instances

---

## Epic Named Release Cadence

Epic releases new software versions twice per year:

| Release | Typical Availability |
|---------|---------------------|
| Spring release | May/June |
| Fall release | November/December |

**Named releases (recent, most recent as of this skill's last update):**
- November 2024 (Tucson)
- May 2025 (Ursa)
- November 2025 (Valley)
- May 2026 (Wren) — pending

**Impact on integrations:**
- Named releases may introduce new FHIR resources, new search parameters, or updated Epic extensions
- Named releases may deprecate parameters or change default behavior
- Epic customers upgrade on their own schedule — not all customers are on the current named release; integrations must handle version variation across a client base
- open.epic.com documentation is version-stamped — when reading documentation, verify it applies to the target Epic version

**Verification pattern:** The FHIR CapabilityStatement (`GET [base]/metadata`) includes the Epic software version. Parse `software.version` to determine the named release.

---

## Care Everywhere and Interoperability

Care Everywhere is Epic's patient record exchange capability. It allows Epic instances to query other Epic and non-Epic systems for patient data when treating a patient.

**Networks:**
- **Carequality:** National framework; most large health systems participate
- **CommonWell:** Alternative national network; fewer Epic customers participate natively (Epic historically preferred Carequality)
- **eHealth Exchange (formerly Nationwide Health Information Network):** Used for government and VA exchange

**TEFCA and QHINs:**
- TEFCA (Trusted Exchange Framework and Common Agreement) is the federal framework governing health information exchange
- QHINs (Qualified Health Information Networks) are the designated exchange operators under TEFCA
- Epic participates in TEFCA through its Care Everywhere infrastructure
- Integrations requiring TEFCA-compliant exchange must use the QHIN-designated exchange endpoints, not direct FHIR calls to an Epic instance

**Integration considerations:**
- Care Everywhere is configured at the Epic instance level — not all features are available at all customers
- Querying Care Everywhere for external patient records is done through Epic's internal workflows, not a public API — third-party apps cannot initiate Care Everywhere queries directly
- FHIR APIs for accessing data retrieved via Care Everywhere are emerging (via TEFCA/QHIN APIs) but are not uniformly available

---

## Common Gotchas

These are patterns that surface repeatedly in the Epic developer community and on third-party healthcare integration blogs. Verify these against current open.epic.com documentation as Epic behavior evolves.

**Registration and credentials:**
- Client credentials are per-customer, per-environment — a single `client_id` does not work across multiple Epic customer instances
- Epic App Orchard review takes weeks to months — budget for this in the project timeline
- Backend service public keys are registered once; there is no automated rotation API — key rotation requires manual re-registration with the Epic customer administrator

**API behavior:**
- Epic returns 404 (not found) for FHIR resources that exist but the requesting user/system does not have access to — this differs from the FHIR spec expectation of 403; do not treat 404 as "this patient has no data"
- Epic's FHIR search may return an empty Bundle (not a 404) when a search produces no results — always check the `total` element in the Bundle
- Epic FHIR `_include` and `_revinclude` are supported but coverage varies by resource — test each `_include` combination explicitly
- Pagination tokens in Epic's `Bundle.link[next]` URLs are time-limited — do not cache them; iterate pages immediately after the initial search response
- Epic's FHIR server returns `OperationOutcome` for errors — always parse and log the OperationOutcome `issue` array; the HTTP status code alone is not sufficient for diagnosis

**Date and time handling:**
- Epic stores dates in the local server timezone; FHIR responses include timezone offsets, but some older interface channels do not — always handle timezone-aware timestamps
- Epic FHIR `dateTime` search parameters support partial dates (`2025`, `2025-03`) — this is useful for range queries without knowing exact timestamps

**Patient matching:**
- Epic patient IDs (`Patient/{id}`) are Epic-internal IDs and are not portable across Epic instances — the same patient at a different Epic customer has a different FHIR Patient ID
- Use `Patient?identifier={system}|{value}` with an MRN or enterprise ID when cross-referencing patients across systems
- Epic's $match operation for patient matching is available but behavior depends on the Epic instance's patient matching configuration

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Using a single global client_id across all Epic customers | Epic credentials are per-customer — a credential registered at one customer does not work at another | Maintain a credential registry keyed by Epic customer/environment |
| Looping individual FHIR resource reads for population-level data | Individual reads are subject to tighter rate limits and are orders of magnitude slower than bulk export | Use bulk FHIR $export for any operation that requires more than a few hundred resources |
| Treating HTTP 404 from Epic as "no data exists" | Epic returns 404 for authorization failures on existing resources | Log the full response including OperationOutcome; investigate 404s that appear on known-existing resources |
| Caching pagination tokens beyond the immediate iteration | Epic pagination tokens expire; cached tokens produce 400/410 errors when reused | Iterate all pages synchronously; do not store next-page URLs for later use |
| Ignoring Epic named release notes when planning integration timelines | Named releases change API behavior; building against the sandbox may not reflect what a customer's older Epic version supports | Check open.epic.com release notes for the target customer's Epic version before finalizing integration design |
| Assuming all Epic customers have the same module configuration | Epic is highly configurable; a FHIR resource available in one customer's environment may not exist in another's | Test integrations against the specific customer's non-production environment, not just the public sandbox |
| Not implementing `_since` parameter for incremental bulk exports | Full patient population exports can take hours and produce gigabytes of data; running a full export daily is wasteful | Implement incremental export using `_since` with the timestamp of the last successful export run |
| Treating Epic's CapabilityStatement as a complete API inventory | Epic's CapabilityStatement reflects the build configuration at a point in time and may not list all supported resources | Verify specific resource and operation support through direct testing and open.epic.com documentation |

---

## Quick Reference

### Epic FHIR Sandbox Base URL

```
https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4
```

### CapabilityStatement (Version Check)

```
GET [base]/metadata
→ response.software.version contains the Epic named release
```

### Bulk Export Initiation

```
GET [base]/Patient/$export?_type=Patient,Observation,Condition&_since=2025-01-01T00:00:00Z
Prefer: respond-async
```

### Poll Export Status

```
GET {Content-Location header from export initiation response}
```

### Named Release to Software Version Mapping (Recent)

| Named Release | Version String Pattern |
|---------------|----------------------|
| November 2024 (Tucson) | Check open.epic.com for exact string |
| May 2025 (Ursa) | Check open.epic.com for exact string |
| November 2025 (Valley) | Check open.epic.com for exact string |

---

## References

- [open.epic.com — Epic Developer Documentation](https://open.epic.com)
- [fhir.epic.com — Epic FHIR Sandbox and API Reference](https://fhir.epic.com)
- [FHIR Bulk Data Access Specification](https://hl7.org/fhir/uv/bulkdata/)
- [USCDI v3 — ONC Official Documentation](https://www.healthit.gov/isa/united-states-core-data-interoperability-uscdi)
- [TEFCA and QHIN Official Resources](https://www.healthit.gov/topic/interoperability/trusted-exchange-framework-and-common-agreement-tefca)
- [Carequality Framework](https://carequality.org/)
- [Epic App Orchard / Connection Hub](https://appmarket.epic.com/)
- See `references/uscdi-v3-epic-detail.md` for detailed USCDI v3 class-to-resource mapping with search parameters
