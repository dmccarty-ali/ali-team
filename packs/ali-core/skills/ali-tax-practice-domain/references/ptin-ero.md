# ali-tax-practice-domain - PTIN and ERO Reference

This reference contains detailed PTIN and ERO requirements, renewal procedures, and compliance specifications.

---

## PTIN Requirements

### Preparer Tax Identification Number

- **Required**: Every paid preparer must have PTIN
- **Renewal**: Annual renewal (typically Oct-Dec for following year)
- **Display**: Must appear on every return prepared for compensation
- **Fee**: ~$30.75 renewal (as of 2024)
- **Application**: IRS website or Form W-12

### Who Needs PTIN?

- Anyone who prepares or assists in preparing federal tax returns for compensation
- Even if only preparing one return
- Volunteers preparing returns for free do NOT need PTIN

### PTIN Format

- Format: P followed by 8 digits (e.g., P12345678)
- Unique to each preparer
- Must be renewed annually
- Cannot be shared or transferred

---

## ERO Requirements

### Electronic Return Originator

**Purpose:** Authorization to e-file returns with the IRS

**Requirements:**
- **EFIN**: Electronic Filing Identification Number
- **Suitability check**: IRS background check required
- **Fingerprinting**: Required for initial application
- **Annual renewal**: EFIN must be renewed each year

### EFIN Application Process

1. **Apply**: Complete IRS Form 8633
2. **Fingerprinting**: Submit fingerprints via approved vendor
3. **Suitability check**: IRS reviews background
4. **Approval**: Receive EFIN (6-8 weeks)
5. **Annual renewal**: Renew before e-filing season

### ERO Responsibilities

- Safeguard EFIN (do not share)
- Maintain client records (3 years minimum for 8879)
- Authenticate client signatures
- Transmit returns accurately
- Report security breaches

---

## Compliance Tracking

```python
class PreparerCompliance:
    preparer_id: str
    ptin: str
    ptin_expiry: datetime
    efin: str
    efin_expiry: datetime
    ptin_renewal_reminder: datetime  # 60 days before expiry
    efin_renewal_reminder: datetime  # 90 days before expiry
    suitability_check_date: datetime
    fingerprint_date: datetime
```

### Renewal Reminders

- **PTIN**: Remind 60 days before expiry (typically Sep 1 for Oct 15 renewal deadline)
- **EFIN**: Remind 90 days before expiry (typically Sep 1 for Dec 31 renewal)
- **Block e-filing**: If either expired, block e-file capability
