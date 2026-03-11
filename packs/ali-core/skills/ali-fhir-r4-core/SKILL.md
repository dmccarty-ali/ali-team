---
name: ali-fhir-r4-core
description: |
  FHIR R4 fundamentals and patterns for healthcare data exchange. Use when:

  PLANNING: Designing FHIR-based integrations, resource models, or API architectures

  IMPLEMENTATION: Writing FHIR parsers, building bundles, implementing FHIR REST APIs

  GUIDANCE: Best practices for resource references, data types, extensions, search

  REVIEW: Validating FHIR conformance, checking resource structure, auditing implementations
---

# FHIR R4 Core Fundamentals

## Overview

Fast Healthcare Interoperability Resources (FHIR) R4 is the HL7 standard for exchanging healthcare data. R4 is the first "normative" release (stable, backward-compatible) and is the dominant version in US healthcare implementations including CMS interoperability rules.

---

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing FHIR-based data exchanges or APIs
- Modeling healthcare data as FHIR resources
- Architecting bundle structures for transactions
- Planning resource references and containment strategies

**Implementation:**
- Parsing or generating FHIR JSON/XML
- Building FHIR bundles (transaction, batch, document, collection)
- Implementing FHIR REST endpoints
- Handling FHIR data types and extensions

**Guidance/Best Practices:**
- Asking about FHIR resource choices for use cases
- How to represent specific clinical/financial data in FHIR
- FHIR search parameter patterns
- Extension vs profile decisions

**Review/Validation:**
- Checking FHIR resource conformance
- Validating bundle structure and references
- Reviewing FHIR implementation correctness
- Auditing US Core profile compliance

---

## Key Principles

- **Resources are atomic units** - Each resource has a logical identity and can be independently retrieved
- **References link resources** - Resources reference each other by URL or logical ID, not embedded copies
- **Bundles aggregate resources** - Use bundles for transactions, documents, or search results
- **Extensions are explicit** - Custom data goes in extensions with proper URLs, never invented elements
- **Profiles constrain** - Implementation guides use profiles to restrict/require elements for specific use cases
- **RESTful by default** - FHIR assumes REST patterns (GET, POST, PUT, DELETE) with predictable URLs
- **JSON preferred** - Modern implementations use JSON; XML is valid but less common
- **US Core is baseline** - For US implementations, US Core profiles are the minimum conformance target

---

## FHIR Resource Structure

### Base Resource Elements

Every FHIR resource has these standard elements:

```json
{
  "resourceType": "Patient",       // Required - identifies the resource type
  "id": "example-123",             // Logical ID within the server
  "meta": {                        // Metadata about the resource
    "versionId": "1",              // Version for optimistic locking
    "lastUpdated": "2024-01-15T10:30:00Z",
    "profile": [                   // Profiles this resource claims conformance to
      "http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient"
    ]
  },
  "text": {                        // Human-readable narrative (recommended)
    "status": "generated",
    "div": "<div xmlns=\"http://www.w3.org/1999/xhtml\">...</div>"
  }
  // ... resource-specific elements follow
}
```

### Resource Categories

| Category | Resources | Purpose |
|----------|-----------|---------|
| **Clinical** | Patient, Encounter, Condition, Observation, DiagnosticReport, Procedure, MedicationRequest, Immunization, AllergyIntolerance | Core clinical data |
| **Financial** | Claim, ExplanationOfBenefit, Coverage, ClaimResponse | Revenue cycle / payer data |
| **Administrative** | Organization, Practitioner, PractitionerRole, Location, HealthcareService, Schedule, Slot | Provider/org directory |
| **Infrastructure** | Bundle, OperationOutcome, Parameters, Binary, Provenance | Exchange mechanics |

---

## Core Clinical Resources

### Patient

The central resource - represents the individual receiving care.

```json
{
  "resourceType": "Patient",
  "id": "pat-12345",
  "identifier": [
    {
      "use": "usual",
      "type": {
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
          "code": "MR"
        }]
      },
      "system": "http://hospital.example.org/mrn",
      "value": "12345678"
    }
  ],
  "name": [
    {
      "use": "official",
      "family": "Smith",
      "given": ["John", "Jacob"]
    }
  ],
  "gender": "male",
  "birthDate": "1970-05-15",
  "address": [
    {
      "use": "home",
      "line": ["123 Main St"],
      "city": "Anytown",
      "state": "CA",
      "postalCode": "12345"
    }
  ]
}
```

**US Core Requirements:**
- At least one identifier
- At least one name
- gender (required)
- birthDate (required if known)

### Encounter

A patient interaction with the healthcare system.

```json
{
  "resourceType": "Encounter",
  "id": "enc-001",
  "status": "finished",
  "class": {
    "system": "http://terminology.hl7.org/CodeSystem/v3-ActCode",
    "code": "IMP",
    "display": "inpatient encounter"
  },
  "type": [
    {
      "coding": [{
        "system": "http://snomed.info/sct",
        "code": "32485007",
        "display": "Hospital admission"
      }]
    }
  ],
  "subject": {
    "reference": "Patient/pat-12345"
  },
  "period": {
    "start": "2024-01-10T08:00:00Z",
    "end": "2024-01-15T11:00:00Z"
  },
  "serviceProvider": {
    "reference": "Organization/org-hospital"
  }
}
```

**Class Codes:**
- AMB = ambulatory/outpatient
- IMP = inpatient
- EMER = emergency
- HH = home health

### Condition

Clinical conditions, problems, diagnoses.

```json
{
  "resourceType": "Condition",
  "id": "cond-001",
  "clinicalStatus": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/condition-clinical",
      "code": "active"
    }]
  },
  "verificationStatus": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/condition-ver-status",
      "code": "confirmed"
    }]
  },
  "category": [
    {
      "coding": [{
        "system": "http://terminology.hl7.org/CodeSystem/condition-category",
        "code": "problem-list-item"
      }]
    }
  ],
  "code": {
    "coding": [
      {
        "system": "http://snomed.info/sct",
        "code": "73211009",
        "display": "Diabetes mellitus"
      },
      {
        "system": "http://hl7.org/fhir/sid/icd-10-cm",
        "code": "E11.9",
        "display": "Type 2 diabetes mellitus without complications"
      }
    ]
  },
  "subject": {
    "reference": "Patient/pat-12345"
  },
  "onsetDateTime": "2020-03-15"
}
```

### Observation

Lab results, vital signs, assessments - the workhorse resource.

```json
{
  "resourceType": "Observation",
  "id": "obs-glucose",
  "status": "final",
  "category": [
    {
      "coding": [{
        "system": "http://terminology.hl7.org/CodeSystem/observation-category",
        "code": "laboratory"
      }]
    }
  ],
  "code": {
    "coding": [{
      "system": "http://loinc.org",
      "code": "2339-0",
      "display": "Glucose [Mass/volume] in Blood"
    }]
  },
  "subject": {
    "reference": "Patient/pat-12345"
  },
  "effectiveDateTime": "2024-01-15T08:30:00Z",
  "valueQuantity": {
    "value": 95,
    "unit": "mg/dL",
    "system": "http://unitsofmeasure.org",
    "code": "mg/dL"
  },
  "referenceRange": [
    {
      "low": {"value": 70, "unit": "mg/dL"},
      "high": {"value": 100, "unit": "mg/dL"}
    }
  ]
}
```

**Category Codes:**
- laboratory - Lab test results
- vital-signs - Vital sign measurements
- social-history - Social history observations
- exam - Physical exam findings

### DiagnosticReport

Groups observations and results from a diagnostic procedure.

```json
{
  "resourceType": "DiagnosticReport",
  "id": "report-001",
  "status": "final",
  "category": [
    {
      "coding": [{
        "system": "http://terminology.hl7.org/CodeSystem/v2-0074",
        "code": "LAB"
      }]
    }
  ],
  "code": {
    "coding": [{
      "system": "http://loinc.org",
      "code": "24323-8",
      "display": "Comprehensive metabolic 2000 panel"
    }]
  },
  "subject": {
    "reference": "Patient/pat-12345"
  },
  "effectiveDateTime": "2024-01-15",
  "issued": "2024-01-15T14:00:00Z",
  "result": [
    {"reference": "Observation/obs-glucose"},
    {"reference": "Observation/obs-sodium"},
    {"reference": "Observation/obs-potassium"}
  ],
  "conclusion": "All values within normal limits."
}
```

---

## Financial Resources

### Coverage

Insurance coverage information.

```json
{
  "resourceType": "Coverage",
  "id": "cov-001",
  "status": "active",
  "type": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/v3-ActCode",
      "code": "HIP",
      "display": "health insurance plan policy"
    }]
  },
  "subscriber": {
    "reference": "Patient/pat-12345"
  },
  "beneficiary": {
    "reference": "Patient/pat-12345"
  },
  "relationship": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/subscriber-relationship",
      "code": "self"
    }]
  },
  "payor": [
    {"reference": "Organization/org-payer"}
  ],
  "class": [
    {
      "type": {
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/coverage-class",
          "code": "group"
        }]
      },
      "value": "GRP12345",
      "name": "Employee Health Plan"
    },
    {
      "type": {
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/coverage-class",
          "code": "plan"
        }]
      },
      "value": "PLN001",
      "name": "Gold PPO"
    }
  ]
}
```

### Claim

Healthcare claim for services rendered.

```json
{
  "resourceType": "Claim",
  "id": "claim-001",
  "status": "active",
  "type": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/claim-type",
      "code": "institutional"
    }]
  },
  "use": "claim",
  "patient": {
    "reference": "Patient/pat-12345"
  },
  "created": "2024-01-16",
  "provider": {
    "reference": "Organization/org-hospital"
  },
  "priority": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/processpriority",
      "code": "normal"
    }]
  },
  "insurance": [
    {
      "sequence": 1,
      "focal": true,
      "coverage": {
        "reference": "Coverage/cov-001"
      }
    }
  ],
  "item": [
    {
      "sequence": 1,
      "productOrService": {
        "coding": [{
          "system": "http://www.ama-assn.org/go/cpt",
          "code": "99213"
        }]
      },
      "servicedDate": "2024-01-15",
      "unitPrice": {
        "value": 150.00,
        "currency": "USD"
      }
    }
  ],
  "total": {
    "value": 150.00,
    "currency": "USD"
  }
}
```

### ExplanationOfBenefit (EOB)

Payer's adjudication result - maps to 835 remittance data.

```json
{
  "resourceType": "ExplanationOfBenefit",
  "id": "eob-001",
  "status": "active",
  "type": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/claim-type",
      "code": "institutional"
    }]
  },
  "use": "claim",
  "patient": {
    "reference": "Patient/pat-12345"
  },
  "created": "2024-01-20",
  "insurer": {
    "reference": "Organization/org-payer"
  },
  "provider": {
    "reference": "Organization/org-hospital"
  },
  "outcome": "complete",
  "item": [
    {
      "sequence": 1,
      "productOrService": {
        "coding": [{
          "system": "http://www.ama-assn.org/go/cpt",
          "code": "99213"
        }]
      },
      "adjudication": [
        {
          "category": {
            "coding": [{
              "system": "http://terminology.hl7.org/CodeSystem/adjudication",
              "code": "submitted"
            }]
          },
          "amount": {"value": 150.00, "currency": "USD"}
        },
        {
          "category": {
            "coding": [{
              "system": "http://terminology.hl7.org/CodeSystem/adjudication",
              "code": "eligible"
            }]
          },
          "amount": {"value": 120.00, "currency": "USD"}
        },
        {
          "category": {
            "coding": [{
              "system": "http://terminology.hl7.org/CodeSystem/adjudication",
              "code": "benefit"
            }]
          },
          "amount": {"value": 96.00, "currency": "USD"}
        }
      ]
    }
  ],
  "total": [
    {
      "category": {
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/adjudication",
          "code": "submitted"
        }]
      },
      "amount": {"value": 150.00, "currency": "USD"}
    },
    {
      "category": {
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/adjudication",
          "code": "benefit"
        }]
      },
      "amount": {"value": 96.00, "currency": "USD"}
    }
  ],
  "payment": {
    "amount": {"value": 96.00, "currency": "USD"}
  }
}
```

---

## Administrative Resources

### Organization

Providers, payers, facilities.

```json
{
  "resourceType": "Organization",
  "id": "org-hospital",
  "identifier": [
    {
      "system": "http://hl7.org/fhir/sid/us-npi",
      "value": "1234567893"
    }
  ],
  "active": true,
  "type": [
    {
      "coding": [{
        "system": "http://terminology.hl7.org/CodeSystem/organization-type",
        "code": "prov",
        "display": "Healthcare Provider"
      }]
    }
  ],
  "name": "General Hospital",
  "telecom": [
    {
      "system": "phone",
      "value": "555-555-5555"
    }
  ],
  "address": [
    {
      "line": ["100 Medical Center Dr"],
      "city": "Anytown",
      "state": "CA",
      "postalCode": "12345"
    }
  ]
}
```

### Practitioner

Individual healthcare provider.

```json
{
  "resourceType": "Practitioner",
  "id": "prac-001",
  "identifier": [
    {
      "system": "http://hl7.org/fhir/sid/us-npi",
      "value": "9876543210"
    }
  ],
  "active": true,
  "name": [
    {
      "family": "Jones",
      "given": ["Sarah"],
      "prefix": ["Dr."],
      "suffix": ["MD"]
    }
  ],
  "qualification": [
    {
      "code": {
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/v2-0360",
          "code": "MD",
          "display": "Doctor of Medicine"
        }]
      }
    }
  ]
}
```

### PractitionerRole

Links practitioners to organizations with specific roles.

```json
{
  "resourceType": "PractitionerRole",
  "id": "role-001",
  "active": true,
  "practitioner": {
    "reference": "Practitioner/prac-001"
  },
  "organization": {
    "reference": "Organization/org-hospital"
  },
  "code": [
    {
      "coding": [{
        "system": "http://snomed.info/sct",
        "code": "59058001",
        "display": "General physician"
      }]
    }
  ],
  "specialty": [
    {
      "coding": [{
        "system": "http://snomed.info/sct",
        "code": "394814009",
        "display": "General practice"
      }]
    }
  ],
  "location": [
    {"reference": "Location/loc-main-clinic"}
  ]
}
```

---

## Bundles

### Bundle Types

| Type | Purpose | Transaction Semantics |
|------|---------|----------------------|
| **transaction** | Atomic batch - all succeed or all fail | Full ACID |
| **batch** | Independent operations - may partially succeed | No transaction |
| **collection** | Grouping resources (no specific semantics) | None |
| **document** | Clinical document with Composition | Document rules |
| **searchset** | Search results | Read-only |
| **message** | Message exchange | Message semantics |

### Transaction Bundle

```json
{
  "resourceType": "Bundle",
  "type": "transaction",
  "entry": [
    {
      "fullUrl": "urn:uuid:patient-001",
      "resource": {
        "resourceType": "Patient",
        "name": [{"family": "Smith", "given": ["John"]}]
      },
      "request": {
        "method": "POST",
        "url": "Patient"
      }
    },
    {
      "fullUrl": "urn:uuid:encounter-001",
      "resource": {
        "resourceType": "Encounter",
        "status": "finished",
        "class": {"code": "AMB"},
        "subject": {
          "reference": "urn:uuid:patient-001"
        }
      },
      "request": {
        "method": "POST",
        "url": "Encounter"
      }
    }
  ]
}
```

**Key Points:**
- fullUrl with urn:uuid: for new resources
- References within bundle use urn:uuid:
- request.method: POST (create), PUT (update), DELETE
- Server resolves UUIDs to actual IDs

### Document Bundle

```json
{
  "resourceType": "Bundle",
  "type": "document",
  "identifier": {
    "system": "http://example.org/documents",
    "value": "doc-12345"
  },
  "timestamp": "2024-01-15T10:00:00Z",
  "entry": [
    {
      "fullUrl": "http://example.org/Composition/comp-001",
      "resource": {
        "resourceType": "Composition",
        "status": "final",
        "type": {
          "coding": [{
            "system": "http://loinc.org",
            "code": "34133-9",
            "display": "Summary of episode note"
          }]
        },
        "subject": {"reference": "Patient/pat-12345"},
        "date": "2024-01-15",
        "author": [{"reference": "Practitioner/prac-001"}],
        "title": "Discharge Summary",
        "section": [
          {
            "title": "Hospital Course",
            "code": {
              "coding": [{
                "system": "http://loinc.org",
                "code": "8648-8"
              }]
            },
            "entry": [
              {"reference": "Condition/cond-001"},
              {"reference": "Procedure/proc-001"}
            ]
          }
        ]
      }
    }
  ]
}
```

---

## References

### Reference Types

| Style | Example | Use When |
|-------|---------|----------|
| **Literal** | `{"reference": "Patient/123"}` | Target exists on same server |
| **Logical** | `{"identifier": {"system": "...", "value": "MRN123"}}` | Cross-system, ID-based lookup |
| **Contained** | `{"reference": "#contained-1"}` | Resource only meaningful in context |
| **URL** | `{"reference": "http://other.org/Patient/456"}` | Cross-server reference |

### Contained Resources

When a resource only makes sense within another resource:

```json
{
  "resourceType": "MedicationRequest",
  "id": "medrx-001",
  "contained": [
    {
      "resourceType": "Medication",
      "id": "med-inline",
      "code": {
        "coding": [{
          "system": "http://www.nlm.nih.gov/research/umls/rxnorm",
          "code": "1049502",
          "display": "Acetaminophen 325 MG"
        }]
      }
    }
  ],
  "status": "active",
  "medicationReference": {
    "reference": "#med-inline"
  },
  "subject": {
    "reference": "Patient/pat-12345"
  }
}
```

**Contained Rules:**
- ID starts with # in reference
- No independent identity - cannot be referenced from outside
- Must be referenced by the containing resource
- Use sparingly - prefer standalone resources

---

## Data Types

### CodeableConcept

Multiple codings for the same concept:

```json
{
  "coding": [
    {
      "system": "http://snomed.info/sct",
      "code": "73211009",
      "display": "Diabetes mellitus"
    },
    {
      "system": "http://hl7.org/fhir/sid/icd-10-cm",
      "code": "E11.9",
      "display": "Type 2 diabetes mellitus without complications"
    }
  ],
  "text": "Type 2 Diabetes"
}
```

### Identifier

```json
{
  "use": "official",
  "type": {
    "coding": [{
      "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
      "code": "MR"
    }]
  },
  "system": "http://hospital.example.org/mrn",
  "value": "12345678",
  "period": {
    "start": "2020-01-01"
  },
  "assigner": {
    "display": "General Hospital"
  }
}
```

### Period and Timing

```json
{
  "period": {
    "start": "2024-01-10T08:00:00Z",
    "end": "2024-01-15T11:00:00Z"
  }
}
```

### Quantity

```json
{
  "value": 95,
  "comparator": "<",
  "unit": "mg/dL",
  "system": "http://unitsofmeasure.org",
  "code": "mg/dL"
}
```

---

## Common Code Systems

| System | URI | Use |
|--------|-----|-----|
| SNOMED CT | `http://snomed.info/sct` | Clinical terms |
| LOINC | `http://loinc.org` | Lab/observation codes |
| ICD-10-CM | `http://hl7.org/fhir/sid/icd-10-cm` | Diagnosis codes |
| CPT | `http://www.ama-assn.org/go/cpt` | Procedure codes |
| RxNorm | `http://www.nlm.nih.gov/research/umls/rxnorm` | Medications |
| NPI | `http://hl7.org/fhir/sid/us-npi` | Provider identifiers |
| UCUM | `http://unitsofmeasure.org` | Units of measure |

---

## FHIR REST API

### URL Patterns

| Operation | HTTP Method | URL Pattern |
|-----------|-------------|-------------|
| Read | GET | `[base]/Patient/123` |
| Search | GET | `[base]/Patient?name=smith` |
| Create | POST | `[base]/Patient` |
| Update | PUT | `[base]/Patient/123` |
| Delete | DELETE | `[base]/Patient/123` |
| History | GET | `[base]/Patient/123/_history` |

### Search Parameters

```
GET /Patient?family=smith&birthdate=1970-05-15
GET /Observation?patient=Patient/123&category=laboratory
GET /Encounter?date=ge2024-01-01&date=lt2024-02-01
GET /Condition?code=http://snomed.info/sct|73211009
```

**Modifiers:**
- `:exact` - Exact string match
- `:contains` - Substring match
- `:missing` - Element missing (true/false)
- `:not` - Negation

**Prefixes (for dates/numbers):**
- `eq` - equals (default)
- `ne` - not equals
- `gt`, `lt`, `ge`, `le` - comparisons

### Include and RevInclude

```
GET /Patient/123?_include=Patient:organization
GET /Encounter?patient=Patient/123&_revinclude=Condition:encounter
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Inventing elements not in spec | Non-conformant, breaks interop | Use extensions with proper URLs |
| Embedding full resources instead of references | Duplicates data, inconsistent updates | Use references; contained only when necessary |
| Hardcoding system URIs as strings | Typos, inconsistency | Define constants for code systems |
| Ignoring meta.profile | Validators won't check conformance | Declare profiles you conform to |
| Using batch when transaction needed | Partial failures leave inconsistent state | Use transaction for atomic operations |
| Putting business logic in contained resources | Hard to query, no independent lifecycle | Use standalone resources with references |
| Missing required US Core elements | Fails conformance checks | Validate against US Core profiles |
| Using display text without coding | Not machine-processable | Always include coding; text is supplementary |

---

## Quick Reference

### US Core Required Elements

**Patient:** identifier, name, gender, birthDate
**Encounter:** status, class, type, subject
**Condition:** clinicalStatus, verificationStatus, category, code, subject
**Observation:** status, category, code, subject

### Common LOINC Codes

| Code | Description |
|------|-------------|
| 8480-6 | Systolic blood pressure |
| 8462-4 | Diastolic blood pressure |
| 8867-4 | Heart rate |
| 8310-5 | Body temperature |
| 29463-7 | Body weight |
| 8302-2 | Body height |
| 2339-0 | Glucose |
| 2823-3 | Potassium |

### Bundle Request Methods

```json
{"request": {"method": "POST", "url": "Patient"}}           // Create
{"request": {"method": "PUT", "url": "Patient/123"}}        // Update
{"request": {"method": "DELETE", "url": "Patient/123"}}     // Delete
{"request": {"method": "GET", "url": "Patient/123"}}        // Read (batch only)
```

---

## References

- [HL7 FHIR R4 Specification](https://hl7.org/fhir/R4/)
- [US Core Implementation Guide](https://www.hl7.org/fhir/us/core/)
- [FHIR Resource Index](https://hl7.org/fhir/R4/resourcelist.html)
- [FHIR Search](https://hl7.org/fhir/R4/search.html)
- [Terminology Module](https://hl7.org/fhir/R4/terminology-module.html)
- [CMS Interoperability Rules](https://www.cms.gov/regulations-and-guidance/guidance/interoperability/index)
