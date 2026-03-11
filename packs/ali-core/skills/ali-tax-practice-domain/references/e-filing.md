# ali-tax-practice-domain - E-Filing Reference

This reference contains detailed e-filing processes, rejection codes, handling workflows, and architecture implementation details.

---

## E-Filing Process

### Transmission Flow

```
Prepare Return → Validate → Transmit → Wait → ACK/NAK → Handle Result
      │              │           │        │        │           │
      │              │           │        │        │           │
      ▼              ▼           ▼        ▼        ▼           │
  Software      Error check   To IRS   24-48hr  Accept or    ├─► If Accept: Complete
  calculates    schema valid  via MeF   wait    Reject       │
                                                              └─► If Reject: Fix & Retransmit
```

### Acknowledgement Types

| Code | Meaning | Action Required |
|------|---------|-----------------|
| **Accepted** | IRS received and accepted | None - return is filed |
| **Rejected** | IRS rejected - errors found | Fix errors, retransmit |
| **Pending** | Still processing | Wait, check again later |

---

## Common Rejection Codes

| Code | Description | Resolution |
|------|-------------|------------|
| IND-031 | SSN already used on another return | Verify SSN, may need paper file |
| IND-032 | Dependent SSN used on another return | Verify dependent, may need paper file |
| IND-181 | AGI doesn't match IRS records | Correct prior year AGI or use IP PIN |
| IND-507 | Name doesn't match SSN | Verify with SSA, correct spelling |
| R0000-902 | Invalid XML structure | Software issue - regenerate return |
| FW2-502 | W-2 EIN not in IRS database | Verify EIN, may need to paper file |

---

## Rejection Handling Workflow

```python
def handle_rejection(rejection_code: str, return_id: str):
    """Standard rejection handling workflow."""

    if rejection_code.startswith('IND-031'):
        # SSN already filed - potential identity theft
        notify_client_possible_id_theft(return_id)
        queue_for_paper_filing(return_id)

    elif rejection_code.startswith('IND-181'):
        # AGI mismatch - common issue
        request_prior_return_from_client(return_id)
        # Or have client request IP PIN from IRS

    elif rejection_code.startswith('R0000'):
        # Technical/XML error
        regenerate_return(return_id)
        resubmit(return_id)

    else:
        # Unknown - queue for manual review
        queue_for_preparer_review(return_id)
```

---

## E-Filing Architecture (TD-020)

### 3-Layer Structure

```
EFilingTransmission (one per return)
    │
    ├── EFilingAcknowledgement (one per jurisdiction: federal + each state)
    │       │
    │       └── EFilingRejection (zero or one per acknowledgement)
    │
    └── [Retry transmissions if rejected]
```

### Transmission Status Flow

```
QUEUED → TRANSMITTING → TRANSMITTED
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
           (wait)                         ERROR
              │                             │
              ▼                             ▼
    Acknowledgement received         Retry or escalate
              │
    ┌─────────┴─────────┐
    ▼                   ▼
 ACCEPTED            REJECTED
    │                   │
    ▼                   ▼
 COMPLETE         Fix & resubmit
```

### Rejection Categories & Escalation

| Category | Escalation Days | Client Action Required |
|----------|-----------------|------------------------|
| data_mismatch | 7 | Yes |
| missing_info | 5 | Sometimes |
| duplicate | 3 | Yes (potential fraud) |
| technical | 1 | No (auto-retry) |
| identity | 10 | Yes (IP PIN needed) |

### Auto-Correctable Errors

Technical errors that can be auto-retried (up to 3 attempts):
- Transmission timeout
- Service unavailable
- Connection reset

---

## References

- [IRS MeF Status Codes](https://www.irs.gov/e-file-providers/ack-and-error-codes) - Acknowledgement Codes
