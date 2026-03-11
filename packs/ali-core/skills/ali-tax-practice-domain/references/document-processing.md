# ali-tax-practice-domain - Document Processing Reference

This reference contains the detailed document processing pipeline, confidence thresholds, and status override procedures.

---

## Document Processing Pipeline

### Pipeline Stages

```
UPLOADING → UPLOADED → SCANNING → CLASSIFYING → EXTRACTING → PROCESSING
     │           │          │           │            │            │
     │           │          │           │            │            │
     ▼           ▼          ▼           ▼            ▼            ▼
  Upload     In S3      Malware      AI/Claude    AI/SurePrep  Validation
  in prog              check                                        │
                          │                                         ▼
                          ▼                               ┌─────────────────┐
                     QUARANTINED                          │ REVIEW_REQUIRED │
                     (if infected)                        └────────┬────────┘
                                                                   │
                                                    ┌──────────────┴──────────────┐
                                                    ▼                             ▼
                                               PROCESSED                      REJECTED
                                                    │                       (if invalid)
                                                    ▼
                                                VERIFIED → ARCHIVED
```

---

## Confidence Thresholds

| Operation | Auto-Approve | Needs Review | Reject |
|-----------|--------------|--------------|--------|
| **Classification** | ≥90% | 70-89% | <70% |
| **Extraction** | ≥85% | 60-84% | <60% |
| **Persona ID** | ≥90% | 70-89% | <70% |
| **AI High Confidence** | ≥80% | ≥60% | <40% |

---

## Status Override (MTG-006)

Documents can have status overridden with audit trail:

### Override Fields

- `status_override`: New status
- `status_override_by`: User who overrode
- `status_override_reason`: Justification
- `status_override_at`: Timestamp
- `status_override_from`: Original status

### Use Cases

- Manual classification when AI fails
- Force processing when confidence low but preparer verified
- Override quarantine for false positive malware detection
- Skip extraction when document is illegible but preparer has source data
