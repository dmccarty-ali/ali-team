---
name: ali-epic-fhir-r4
description: |
  Epic-specific FHIR R4 patterns and deviations from the base FHIR standard. Use when:

  PLANNING: Designing Epic FHIR integrations, evaluating Epic-specific search parameters,
  planning Epic FHIR R4 vs STU3 migration, choosing which Epic FHIR resources to use

  IMPLEMENTATION: Writing code against open.epic.com FHIR R4 APIs, handling Epic-specific
  extensions, implementing Epic FHIR error handling, setting up Epic sandbox environments

  GUIDANCE: Asking about Epic FHIR deviations from base spec, Epic-supported search
  parameters, Epic FHIR versioning and release cadence, Epic sandbox test patients

  REVIEW: Validating FHIR resource usage against open.epic.com specs, checking Epic-specific
  search parameter support, auditing Epic FHIR error handling, reviewing Epic API pagination
---

# Epic FHIR R4 Patterns

## Overview

Epic's FHIR R4 implementation deviates from the base HL7 standard in significant ways. Epic supports a curated subset of resources, uses named release versioning, has its own extension namespace, and enforces pagination and rate limits that differ from standard FHIR server expectations. This skill adds the Epic-specific layer on top of ali-fhir-r4-core — do not duplicate base FHIR knowledge here.

---

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing an integration against open.epic.com or fhir.epic.com APIs
- Choosing between Epic FHIR R4 and Epic FHIR STU3 for a new integration
- Evaluating which Epic FHIR resources and search parameters are available for a use case
- Planning Epic sandbox setup and test patient strategy

**Implementation:**
- Writing code that calls Epic FHIR R4 endpoints
- Handling Epic OperationOutcome error responses
- Parsing Epic-specific extensions on standard FHIR resources
- Implementing pagination for Epic FHIR search results
- Working with the Epic Sandbox (fhir.epic.com) environment

**Guidance/Best Practices:**
- Asking which FHIR search parameters Epic actually supports for a given resource
- Asking about Epic's named release versioning (e.g., "August 2025" release)
- Asking how Epic FHIR behavior differs from the base spec
- Asking about known Epic FHIR gotchas, limitations, or deviations

**Review/Validation:**
- Reviewing code that calls Epic FHIR endpoints for correctness
- Checking whether a search parameter combination is supported by Epic
- Auditing error handling against Epic's OperationOutcome patterns
- Verifying that extensions are being read from the correct Epic namespace

---

## When NOT to Use This Skill

- For base FHIR R4 questions not specific to Epic — use ali-fhir-r4-core instead
- For HL7 v2 questions (ADT, ORM, ORU interfaces) — use ali-epic-hl7v2 or ali-hl7-v2-core
- For SMART on FHIR / OAuth 2.0 / authentication questions — use ali-epic-smart-auth
- For non-FHIR Epic topics (Epic module configuration, Clarity reporting, operational workflows, MyChart administration) — this skill covers integration APIs only
- For CDS Hooks or FHIRcast — use ali-epic-smart-auth or ali-epic-integration-patterns

---

## Key Principles

- **Epic supports a subset, not all of FHIR R4** — never assume a resource or search parameter works; verify against open.epic.com before implementing
- **All endpoints require authentication** — there are no anonymous Epic FHIR calls; OAuth 2.0 via SMART on FHIR is mandatory
- **Named releases, not semantic versioning** — Epic uses calendar-named releases ("August 2025", "February 2026") that may add or remove support; check release notes when upgrading
- **Epic extensions are in a documented namespace** — extensions use `http://open.epic.com/FHIR/StructureDefinition/extension/` as the base URL
- **Pagination is not optional** — Epic returns Bundles with `next` links for search results; clients that do not follow next links will silently miss records
- **OperationOutcome is the error contract** — parse `issue[*].code` and `issue[*].diagnostics`, not HTTP status codes alone
- **Sandbox and production behave differently** — sandbox test patients are curated; sandbox does not reflect the full production resource set
- **STU3 is legacy** — Epic continues to support STU3 for backwards compatibility but new integrations must target R4; some resources exist in STU3 that have no R4 equivalent yet (and vice versa)

---

## Epic FHIR R4 Base URL Pattern

Epic's FHIR R4 base URL follows this convention:

```
https://{organization-fhir-server}/api/FHIR/R4/
```

The public sandbox base URL is:

```
https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/
```

Every request to any endpoint under this base requires a valid Bearer token from the SMART on FHIR authorization flow. There is no unauthenticated access, including read operations.

**Organization-specific base URLs** are provided during Epic Go-Live. They are not guessable from organization name. During development, always use the fhir.epic.com sandbox URL.

---

## Epic FHIR Versioning Model

Epic releases FHIR API updates on a named-release calendar, not semantic versioning:

| Release Name | Approximate Date | Notes |
|--------------|-----------------|-------|
| August 2024 | Q3 2024 | Stable baseline for current integrations |
| February 2025 | Q1 2025 | Added bulk data improvements |
| August 2025 | Q3 2025 | Current stable release |
| February 2026 | Q1 2026 | Planned; check open.epic.com for changes |

**Version header:** Clients can request a specific Epic release by adding:

```http
Epic-Client-ID: {your-client-id}
```

The response includes `X-Epic-Release` header indicating the server's current release. Clients should log this value and alert on unexpected changes.

**Breaking change cadence:** Epic does not remove support mid-release but may deprecate parameters in one release and remove them two releases later. Monitor `Sunset` headers in responses and subscribe to Epic's developer release notes.

**FHIR version identifier:** Epic's R4 implementation reports `fhirVersion: 4.0.1` in the CapabilityStatement. The STU3 endpoint reports `fhirVersion: 3.0.1`.

---

## Supported Resources and What to Verify

Epic does not support every FHIR R4 resource. The following are commonly supported and stable, but always verify for a specific organization's Epic version:

| Resource | Read | Search | Create/Update | Notes |
|----------|------|--------|---------------|-------|
| Patient | Yes | Yes | No (org-controlled) | Search by MRN, name, DOB, SSN |
| Encounter | Yes | Yes | No | Inpatient, outpatient, emergency |
| Observation | Yes | Yes | Limited | Labs, vitals; category param required |
| Condition | Yes | Yes | Limited | Problem list; chronic vs acute |
| MedicationRequest | Yes | Yes | Limited | Active prescriptions |
| AllergyIntolerance | Yes | Yes | No | Active allergies |
| DocumentReference | Yes | Yes | Yes | Clinical notes via FHIR |
| Coverage | Yes | Yes | No | Insurance coverage |
| Appointment | Yes | Yes | Limited | Scheduling (limited write) |
| Immunization | Yes | Yes | No | Vaccination history |
| Procedure | Yes | Yes | No | Completed procedures |
| DiagnosticReport | Yes | Yes | No | Grouped lab results |

**Resources not supported or limited in R4 (as of August 2025):**

- `Device` — limited support; check capability statement
- `Goal` — read only in most installations
- `CarePlan` — limited; varies by Epic configuration
- `Consent` — not exposed via FHIR R4 in standard installs
- `Contract` — not supported

**Always check the CapabilityStatement** for the specific Epic server before building:

```
GET https://{fhir-base}/metadata
```

---

## Supported Search Parameters by Resource

Epic supports a curated subset of FHIR search parameters. Parameters not listed here will return an error or empty results.

### Patient

```
GET /Patient?identifier={system}|{value}      # MRN lookup (required system prefix)
GET /Patient?family={name}&birthdate={date}   # Name + DOB combo
GET /Patient?_id={logical-id}                 # Direct ID lookup
GET /Patient?identifier=urn:oid:2.16.840.1.113883.4.1|{ssn}  # SSN lookup
```

Epic requires the `system` prefix in identifier searches (e.g., `http://hospital.org/mrn|12345`). A bare value search without a system prefix may fail or return unexpected results.

### Observation

```
GET /Observation?patient={id}&category=laboratory
GET /Observation?patient={id}&category=vital-signs
GET /Observation?patient={id}&code={loinc-code}
GET /Observation?patient={id}&date=ge{date}&date=le{date}
```

The `category` parameter is required for Observation searches — Epic will not return all observations for a patient without it. Searching by `code` alone (without `patient`) is not supported.

### Condition

```
GET /Condition?patient={id}
GET /Condition?patient={id}&category=problem-list-item
GET /Condition?patient={id}&clinical-status=active
```

### MedicationRequest

```
GET /MedicationRequest?patient={id}
GET /MedicationRequest?patient={id}&status=active
GET /MedicationRequest?patient={id}&intent=order
```

### DocumentReference

```
GET /DocumentReference?patient={id}
GET /DocumentReference?patient={id}&type={loinc-code}
GET /DocumentReference?patient={id}&date=ge{date}
GET /DocumentReference/{id}/$docref    # Retrieve document content
```

---

## Epic FHIR Extensions

Epic uses a documented extension namespace. Extensions appear on standard resources and carry Epic-specific data not in the base FHIR spec.

**Namespace:** `http://open.epic.com/FHIR/StructureDefinition/extension/`

### Common Patient Extensions

```json
{
  "extension": [
    {
      "url": "http://open.epic.com/FHIR/StructureDefinition/extension/legal-sex",
      "valueCodeableConcept": {
        "coding": [{
          "system": "http://open.epic.com/FHIR/StructureDefinition/legal-sex",
          "code": "male",
          "display": "Male"
        }]
      }
    },
    {
      "url": "http://open.epic.com/FHIR/StructureDefinition/extension/preferred-pronoun",
      "valueString": "he/him"
    }
  ]
}
```

### Common Encounter Extensions

```json
{
  "extension": [
    {
      "url": "http://open.epic.com/FHIR/StructureDefinition/extension/encounter-type",
      "valueString": "Inpatient"
    },
    {
      "url": "http://open.epic.com/FHIR/StructureDefinition/extension/is-ed-encounter",
      "valueBoolean": false
    }
  ]
}
```

**Parsing rule:** Always check for the presence of an extension before accessing it. Absence of an extension is normal — do not treat a missing extension as an error. Epic only populates extensions when the underlying data exists in the EHR.

---

## Pagination Pattern

Epic FHIR search results always return a Bundle. When results exceed a page, a `next` link is included.

```json
{
  "resourceType": "Bundle",
  "type": "searchset",
  "total": 142,
  "link": [
    {
      "relation": "self",
      "url": "https://fhir.epic.com/.../Observation?patient=abc&category=laboratory&_count=50"
    },
    {
      "relation": "next",
      "url": "https://fhir.epic.com/.../Observation?patient=abc&category=laboratory&_count=50&_page=2"
    }
  ],
  "entry": [...]
}
```

**Implementation pattern:**

```python
def fetch_all_pages(initial_url: str, headers: dict) -> list[dict]:
    """Fetch all pages of an Epic FHIR search result."""
    resources = []
    url = initial_url

    while url:
        response = requests.get(url, headers=headers)
        response.raise_for_status()
        bundle = response.json()

        for entry in bundle.get("entry", []):
            resources.append(entry["resource"])

        # Find next page link
        next_link = next(
            (link["url"] for link in bundle.get("link", [])
             if link["relation"] == "next"),
            None
        )
        url = next_link

    return resources
```

**Default page size:** Epic defaults to 30 records per page for most resources. Use `_count` to request up to 100:

```
GET /Observation?patient={id}&category=laboratory&_count=100
```

Values above 100 are silently clamped to 100 by Epic.

---

## Error Response Handling

Epic returns OperationOutcome resources for all API errors. Do not rely on HTTP status codes alone.

```json
{
  "resourceType": "OperationOutcome",
  "issue": [
    {
      "severity": "error",
      "code": "not-found",
      "details": {
        "coding": [{
          "system": "urn:oid:1.2.840.114350.1.13.0.1.7.2.657369",
          "code": "4100",
          "display": "Patient not found"
        }]
      },
      "diagnostics": "Patient/xyz was not found or access was denied."
    }
  ]
}
```

**Common Epic error codes:**

| HTTP Status | OperationOutcome code | Meaning |
|-------------|----------------------|---------|
| 400 | `invalid` | Malformed request, unsupported search parameter |
| 401 | `security` | Missing or expired token |
| 403 | `forbidden` | Insufficient SMART scopes |
| 404 | `not-found` | Resource does not exist or patient access denied |
| 422 | `processing` | Valid FHIR but business rule violation |
| 429 | `throttled` | Rate limit exceeded |
| 500 | `exception` | Epic server error; retry with backoff |

**Parsing pattern:**

```python
def parse_epic_error(response: requests.Response) -> str:
    """Extract a readable error message from an Epic OperationOutcome."""
    try:
        body = response.json()
        if body.get("resourceType") == "OperationOutcome":
            issues = body.get("issue", [])
            if issues:
                return issues[0].get("diagnostics", issues[0].get("code", "Unknown error"))
    except (ValueError, KeyError):
        pass
    return f"HTTP {response.status_code}: {response.text[:200]}"
```

---

## Rate Limiting

Epic enforces rate limits at the client application level. As of August 2025:

- **Standard rate limit:** 100 requests per minute per client application ID
- **Burst allowance:** Short bursts above the limit are allowed; sustained overages trigger 429 responses
- **Bulk data exemption:** Bulk FHIR export (`$export`) operations are not subject to per-minute limits but are subject to concurrency limits (typically 1 concurrent export per client)
- **Retry-After header:** Epic includes `Retry-After` (seconds) in 429 responses

**Retry pattern:**

```python
import time

def call_with_retry(url: str, headers: dict, max_retries: int = 3) -> dict:
    """Call Epic FHIR endpoint with rate-limit retry."""
    for attempt in range(max_retries):
        response = requests.get(url, headers=headers)

        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", 60))
            if attempt < max_retries - 1:
                time.sleep(retry_after)
                continue
            raise RateLimitError(f"Rate limit exceeded after {max_retries} attempts")

        response.raise_for_status()
        return response.json()

    raise RuntimeError("Retry loop exhausted without success or exception")
```

---

## Epic Sandbox Environment

**Sandbox base URL:** `https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/`

**Sandbox OAuth URL:** `https://fhir.epic.com/interconnect-fhir-oauth/oauth2/`

### App Registration

1. Create an account at `fhir.epic.com`
2. Navigate to Build Apps and create a new application
3. Choose SMART on FHIR (standalone launch or EHR launch)
4. Configure redirect URIs and requested scopes
5. Receive a non-production Client ID
6. Use the non-production Client ID against the sandbox only — production requires a separate review

### Sandbox Test Patients

Epic provides curated test patients in the sandbox. As of February 2025, key test patients include:

| Patient Name | FHIR ID | Scenarios Covered |
|--------------|---------|-------------------|
| Camila Lopez | `eD0SBrCCJbW0lMHF9EWS5OQ3` | Standard adult with labs, meds, conditions |
| Derrick Lin | `eq081-VQEgP8drUUqCWzHfw3` | Pediatric patient |
| Elijah Barber | `eGg6UCJQXF8cNzEumJPqo3g3` | Encounter history |
| Jason Argonaut | `Tbt3KuCY0B5PSrJvCu2j-PlK.aiHsu2xUjUM8bWpue9vwB3Cex0jaxFg0lq1dBMkjE53GQA3` | Used in official Epic tutorials |

**Important sandbox caveats:**
- Sandbox data is reset periodically; do not depend on created resources persisting across resets
- The sandbox does not implement all Epic-specific extensions present in production
- Rate limits in the sandbox are more lenient than production — do not use sandbox performance as a production estimate
- Some resources available in production are absent or stubbed in sandbox
- Bulk FHIR export is functional in sandbox but uses synthetic data only

---

## Epic R4 vs STU3 Key Differences

| Aspect | STU3 | R4 |
|--------|------|----|
| Medication resources | MedicationOrder | MedicationRequest |
| Condition status | No clinicalStatus coding system | `http://terminology.hl7.org/CodeSystem/condition-clinical` |
| Patient.animal | Supported | Removed |
| Bundle.entry.request | Different method semantics | Standardized |
| Search `_include` | Partial support | Expanded support |
| Coverage | Basic | Enhanced with class hierarchy |

**Migration note:** STU3 endpoints remain available at `/api/FHIR/STU3/` but Epic has stated a long-term deprecation trajectory. All new integrations must use R4. When migrating existing STU3 code, the most common breaking change is `MedicationOrder` → `MedicationRequest` and Condition coding system updates.

---

## DocumentReference — Clinical Notes Pattern

Clinical notes are the most common use case for DocumentReference in Epic. The resource pattern uses a two-step fetch:

**Step 1: Search for note metadata**

```
GET /DocumentReference?patient={id}&type=34133-9
```

LOINC 34133-9 is "Summary of episode note" (discharge summary). Common note LOINC codes:

| LOINC | Note Type |
|-------|-----------|
| 34133-9 | Discharge summary |
| 11506-3 | Progress note |
| 34117-2 | History and physical |
| 18842-5 | Discharge note |
| 11488-4 | Consultation note |

**Step 2: Retrieve note content**

The `content[0].attachment.url` in the DocumentReference contains the actual note URL. Call it to retrieve the content:

```python
doc_ref = fetch_resource("DocumentReference", doc_id)
content_url = doc_ref["content"][0]["attachment"]["url"]
note_response = requests.get(content_url, headers=auth_headers)
# Returns text/plain or text/html depending on Epic configuration
```

---

## Coverage — Insurance Data Pattern

Coverage in Epic reflects the patient's insurance enrollment. Common patterns:

```
GET /Coverage?patient={id}
GET /Coverage?patient={id}&status=active
```

Epic's Coverage resource includes a `class` array with group number and plan identifiers that do not exist in the base FHIR spec. Always check both `class[type.code == 'group'].value` and `class[type.code == 'plan'].value`.

The `payor` reference points to an Organization resource representing the insurance company, but Epic may use logical references (identifier-based) rather than literal references (Organization/{id}) in some installations.

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Assuming all FHIR R4 search params work | Epic supports a subset; unsupported params return errors or empty results | Check open.epic.com API docs for each resource before implementing |
| Not following pagination next links | Silently retrieves only the first 30 records | Always loop on `link[relation=next]` until absent |
| Using DSTU2 or STU3 endpoints for new integrations | Legacy paths; DSTU2 is decommissioned in most installs | Target `/api/FHIR/R4/` for all new work |
| Ignoring Epic extensions | Missing Epic-specific data (legal sex, preferred pronouns, encounter type) | Parse extensions from the `http://open.epic.com/FHIR/StructureDefinition/extension/` namespace |
| Treating HTTP 403 and 404 as equivalent | 403 = wrong scopes; 404 = patient/resource not found or access denied | Parse OperationOutcome to distinguish and handle each case |
| Searching Observation without `category` | Epic requires category; omitting it returns an error or empty bundle | Always include `category=laboratory` or `category=vital-signs` |
| Searching Patient by identifier without system prefix | Produces unexpected results or errors | Always use `identifier={system}|{value}` format |
| Hardcoding sandbox test patient IDs | Sandbox is periodically reset; IDs may change | Store test patient IDs in environment configuration, not code |
| Not checking CapabilityStatement | Assuming resource support; production differs from sandbox | Fetch `/metadata` and cache the CapabilityStatement at integration startup |
| Trusting sandbox rate limits as production estimates | Sandbox rate limits are more lenient | Design against documented production rate limits (100 req/min) |
| Not logging `X-Epic-Release` header | Blind to server-side release upgrades that may break behavior | Log the header per request; alert on unexpected changes |

---

## CapabilityStatement Usage

The CapabilityStatement (returned by `GET {base}/metadata`) is the authoritative source of truth for what a specific Epic installation supports. Epic configurations differ across organizations — a resource or parameter that works in the sandbox or at one hospital may not work at another.

**Fetch and cache at startup:**

```python
def load_capability_statement(base_url: str, headers: dict) -> dict:
    """
    Fetch the CapabilityStatement and cache it.
    Call once at integration startup; re-fetch on 400 errors
    caused by unsupported parameters.
    """
    response = requests.get(f"{base_url}/metadata", headers=headers)
    response.raise_for_status()
    return response.json()

def get_supported_search_params(cap_stmt: dict, resource_type: str) -> set[str]:
    """Extract supported search parameter names for a resource type."""
    for rest in cap_stmt.get("rest", []):
        for resource in rest.get("resource", []):
            if resource.get("type") == resource_type:
                return {
                    sp["name"]
                    for sp in resource.get("searchParam", [])
                }
    return set()
```

**What to check in the CapabilityStatement before implementing:**

- `rest[0].resource[?type=Patient].interaction` — which CRUD operations are supported
- `rest[0].resource[?type=Observation].searchParam` — which search parameters are available
- `rest[0].security.service[0].coding[0].code` — confirms SMART on FHIR is the auth mechanism
- `fhirVersion` — confirms R4 (4.0.1) vs STU3 (3.0.1)

---

## UserWeb-Gated Topics

Some Epic FHIR knowledge is behind Epic's UserWeb portal, which requires an active Epic customer or partner account. When asked about these areas, state that the answer requires UserWeb access and cannot be confirmed from public documentation.

**Topics that require UserWeb:**

- Epic-specific error codes beyond the documented public set (Epic has an internal error code reference)
- Epic Chronicles database mappings to FHIR (Clarity reporting field to FHIR element mappings)
- Epic module-specific FHIR configuration (how a specific Epic module exposes FHIR data depends on configuration, not public spec)
- Epic Showroom and App Orchard submission requirements and review criteria
- Epic's internal testing and certification process for production access
- Organization-specific endpoint URLs (each Epic customer has a unique FHIR base URL)
- Hyperdrive SDK integration patterns beyond what is documented on open.epic.com

**Response pattern when a question requires UserWeb:**

> "This specific detail about [topic] is documented in Epic's UserWeb portal, which requires an active Epic customer or partner account. I can provide information from public documentation at open.epic.com and fhir.epic.com, but I cannot confirm [specific claim] without UserWeb access. If you have UserWeb access, look in [general area]."

---

## Quick Reference

### Common URLs

```
# Sandbox base
https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/

# Capability statement
GET {base}/metadata

# Patient read
GET {base}/Patient/{id}

# Patient search by MRN
GET {base}/Patient?identifier=http://hospital.org/mrn|{mrn}

# Lab results
GET {base}/Observation?patient={id}&category=laboratory

# Active medications
GET {base}/MedicationRequest?patient={id}&status=active

# Problem list
GET {base}/Condition?patient={id}&category=problem-list-item

# Allergies
GET {base}/AllergyIntolerance?patient={id}

# Insurance coverage
GET {base}/Coverage?patient={id}&status=active

# Clinical notes
GET {base}/DocumentReference?patient={id}&type={loinc-code}
```

### Epic Extension Namespace

```
http://open.epic.com/FHIR/StructureDefinition/extension/
```

### Key HTTP Headers

```http
Authorization: Bearer {access_token}
Accept: application/fhir+json
Content-Type: application/fhir+json
Epic-Client-ID: {your-client-id}
```

---

## References

- [Epic FHIR R4 API Documentation](https://fhir.epic.com/Documentation)
- [open.epic.com FHIR R4 Resource List](https://open.epic.com/Clinical/Patient)
- [Epic Sandbox Setup Guide](https://fhir.epic.com/Documentation?docId=developmentapp)
- [Epic FHIR Release Notes](https://open.epic.com/Interface/HL7FHIR)
- [HL7 FHIR R4 Base Specification](https://hl7.org/fhir/R4/)
- [US Core Implementation Guide](https://www.hl7.org/fhir/us/core/)
- [SMART on FHIR App Launch](https://www.hl7.org/fhir/smart-app-launch/)
