# ali-tax-practice-domain - Identity Verification Reference

This reference contains detailed identity verification requirements, tier specifications, and Persona integration details.

---

## Identity Verification

### IRS Requirements

Tax preparers must verify client identity before filing. Levels of verification:

| Level | Method | When Used |
|-------|--------|-----------|
| **Basic** | Prior year AGI match | Returning clients |
| **Document** | Government ID + SSN card | New clients, standard |
| **Enhanced** | Video call + document verification | High-risk, remote clients |

### Verification Methods

**Prior Year AGI:**
```
- Compare client-provided AGI to IRS records
- Used for IRS e-file identity confirmation
- Client must know prior year AGI or obtain IP PIN
```

**Document Verification:**
```
- Government-issued photo ID (driver's license, passport)
- Social Security card or SSA-1099
- Proof of address (utility bill, bank statement)
- Keep copies in client file
```

**Video Verification (Persona Integration):**
```
- Live video session with ID verification
- Document capture and facial match
- Used for remote clients without in-person meeting
```

---

## Multi-Tier Identity Verification

### Verification Tiers

| Tier | Name | Requirements | Use Case |
|------|------|--------------|----------|
| **TIER_0** | Unverified | None | Initial state |
| **TIER_1** | Returning Client | Name + SSN-4 + DOB + Prior Year AGI | Existing clients |
| **TIER_2** | New Client | Email + Phone + Gov ID + Persona Check | Standard new clients |
| **TIER_3** | High-Risk | Enhanced Persona + Manual Review | Flagged accounts |

### Verification Components

```python
class IdentityVerification:
    # Component statuses
    email_verified: bool
    phone_verified: bool           # SMS or voice code
    document_verified: bool        # Government ID scan
    persona_verified: bool         # Persona inquiry result
    prior_year_agi_verified: bool  # IRS AGI match

    # Persona details
    persona_inquiry_id: str
    persona_confidence: float      # 0.0-1.0
    selfie_match: bool
    document_authentic: bool
    data_consistent: bool
    id_not_expired: bool
```

### Phone Verification Limits

| Parameter | Value |
|-----------|-------|
| Code expiry | 10 minutes |
| Max attempts | 3 per code |
| Rate limit | 3 codes/hour |

---

## Persona Integration

### Live Document Verification

- Live document capture and verification
- Facial match (selfie to ID photo)
- Confidence scoring with auto-approval thresholds
- Risk signal detection

### Auto-Approval Thresholds

| Operation | Auto-Approve | Needs Review | Reject |
|-----------|--------------|--------------|--------|
| **Persona ID** | ≥90% | 70-89% | <70% |

### Verification Fields

- `persona_inquiry_id`: Unique Persona inquiry reference
- `persona_confidence`: Confidence score (0.0-1.0)
- `selfie_match`: Did selfie match ID photo?
- `document_authentic`: Was document authentic?
- `data_consistent`: Are all fields consistent?
- `id_not_expired`: Is ID still valid?
