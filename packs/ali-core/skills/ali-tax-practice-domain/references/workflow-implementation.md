# ali-tax-practice-domain - Workflow Implementation Reference

This reference contains the detailed workflow state machine, state transitions, progress tracking, priority levels, and exception handling.

---

## Workflow State Machine

### 17 Workflow States

The tax return workflow uses a defined state machine with valid transitions:

```
INTAKE → DOCUMENTS_PENDING → READY_FOR_PREP → AI_ANALYSIS → IN_PREP
                                                              │
    ┌─────────────────────────────────────────────────────────┘
    ▼
READY_FOR_REVIEW → IN_REVIEW → APPROVED → PENDING_SIGNATURE
                       │                         │
                       ▼                         ▼
                 REVISIONS_NEEDED         READY_TO_FILE → FILED
                       │                                    │
                       └──────────────────┐                 ▼
                                          │            ACCEPTED → COMPLETE
                                          │                │
                                          ▼                ▼
                              PENDING_CLIENT_RESPONSE   REJECTED
                                          │                │
                                          └────────────────┘

Additional states: NEEDS_REVIEW (exception handling)
```

---

## State Progress Mapping

| State | Progress | Description |
|-------|----------|-------------|
| INTAKE | 5% | Initial client contact, engagement |
| DOCUMENTS_PENDING | 10% | Waiting for documents |
| READY_FOR_PREP | 20% | Documents complete, awaiting assignment |
| AI_ANALYSIS | 25% | AI preliminary analysis running |
| IN_PREP | 40% | Preparer actively working |
| READY_FOR_REVIEW | 60% | Prep complete, awaiting reviewer |
| IN_REVIEW | 70% | Reviewer actively checking |
| APPROVED | 75% | Review passed |
| PENDING_SIGNATURE | 80% | Waiting for 8879 signature |
| READY_TO_FILE | 85% | Signed, ready to transmit |
| FILED | 90% | Transmitted to IRS |
| ACCEPTED | 95% | IRS acknowledged |
| COMPLETE | 100% | Delivered to client, archived |

---

## Priority Levels & Rush Fees

| Priority | Turnaround | Rush Fee |
|----------|------------|----------|
| normal | Standard | $0 |
| rush_48 | 48 hours | $150 |
| rush_24 | 24 hours | $300 |

---

## Workflow Exceptions

### Exception Types

| Type | Description |
|------|-------------|
| classification_failed | AI couldn't classify document |
| extraction_failed | AI couldn't extract fields |
| validation_error | Data doesn't pass validation |
| low_confidence | AI confidence below threshold |
| missing_required | Required document not uploaded |
| duplicate_detected | Duplicate document in batch |
| manual_escalation | Preparer flagged for review |

### Exception Aging

| Parameter | Value |
|-----------|-------|
| Aging threshold | 24 hours |
| Alert trigger | Exception unresolved past threshold |
| Escalation | Manager notification |
