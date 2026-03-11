# ali-tax-practice-domain - Data Retention Reference

This reference contains detailed IRS retention requirements and secure disposal procedures.

---

## Data Retention

### IRS Requirements

| Document Type | Retention Period |
|---------------|------------------|
| Tax returns (copy) | 7 years |
| Form 8879 | 3 years from due date |
| Form 7216 consent | Duration of consent + 3 years |
| Client workpapers | 7 years |
| Engagement letters | 7 years |
| ID verification docs | 7 years |
| W-2s, 1099s (client copies) | 7 years |
| Supporting receipts | 7 years |
| Bank statements | 7 years |
| Investment records | 7 years (or until sold + 7) |
| Property records | 7 years after disposal |

---

## Retention Rationale

**Why 7 years?**
- IRS can audit returns up to 3 years back (standard)
- IRS can audit up to 6 years for substantial underreporting (>25% income)
- 7 years provides safe buffer
- Fraud or no filing = no statute of limitations

**Form 8879 - Why 3 years?**
- IRS regulation requires 3 years from return due date
- Shorter than general 7-year rule
- Must prove authorization to e-file

**Form 7216 - Why consent duration + 3?**
- Must retain for duration of consent period
- Plus 3 years after consent expires
- Proves consent was obtained for any disclosure

---

## Secure Disposal

After retention period:

### Physical Documents
- **Shred**: Cross-cut shredder (minimum)
- **Document**: Log disposal date and method
- **Witness**: Two-person disposal for high-sensitivity
- **Certificate**: Obtain destruction certificate if using service

### Digital Files
- **Secure delete**: Use secure deletion tools (overwrite)
- **Backup purge**: Remove from all backups
- **Cloud deletion**: Verify cloud provider permanent deletion
- **Log**: Document deletion date and method
- **Verify**: Confirm files unrecoverable

---

## Retention Schedule Automation

```python
class RetentionSchedule:
    document_type: str
    retention_years: int
    creation_date: datetime
    disposal_eligible_date: datetime
    disposal_completed: bool
    disposal_date: datetime
    disposal_method: str
    disposal_by: str
```

### Automated Checks

- Weekly scan for documents past retention
- Quarterly disposal batches
- Annual retention policy review
- Audit trail of all disposals
