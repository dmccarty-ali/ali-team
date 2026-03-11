---
name: ali-tax-practice-domain
description: |
  Tax practice operations and IRS compliance for Aliunde Tax Practice AI. Use when:

  PLANNING: Designing tax preparation workflows, planning client intake processes,
  architecting document handling, considering compliance requirements

  IMPLEMENTATION: Building tax return workflows, implementing e-filing integration,
  creating identity verification flows, handling document processing

  GUIDANCE: Asking about IRS requirements, Circular 230, WISP, form requirements,
  e-filing process, rejection handling, compliance deadlines

  REVIEW: Checking compliance with IRS regulations, validating workflow completeness,
  reviewing identity verification requirements, auditing data retention

  Do NOT use for:
  - Infrastructure/DevOps (use ali-infrastructure, ali-aws-architecture, ali-terraform)
  - Database design (use ali-database-developer, ali-aurora-postgresql)
  - General backend development (use ali-backend-developer)
  - Testing strategies (use ali-test-developer, ali-testing)
  - Security implementation (use ali-secure-coding)
  - API design (use ali-backend-developer, ali-external-integrations)
  - Document storage/S3 (use ali-aws-architecture)

  Use ONLY for tax-specific domain knowledge: IRS compliance, tax return lifecycle,
  e-filing, preparer requirements, client consent, identity verification for tax purposes.
---

# Tax Practice Domain

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing tax preparation workflows
- Planning client intake and onboarding
- Architecting document collection and processing
- Considering IRS compliance requirements

**Implementation:**
- Building tax return preparation features
- Implementing e-filing transmission
- Creating identity verification flows
- Building document upload and processing

**Guidance/Best Practices:**
- Asking about IRS regulations (Circular 230, WISP)
- Asking about form requirements (7216, 8879)
- Asking about e-filing process and rejection handling
- Asking about preparer requirements (PTIN, ERO)

**Review/Validation:**
- Checking compliance with IRS requirements
- Validating workflow completeness
- Reviewing consent and authorization flows
- Auditing data retention policies

---

## Key Principles

- **Client consent required**: Form 7216 before any disclosure to third parties
- **Identity verification before filing**: IRS requires preparer to verify client identity
- **PTIN on every return**: Every paid preparer must have valid PTIN
- **Retain 7 years**: Tax records and supporting documents
- **E-file acknowledgements**: Track acceptance/rejection for every transmission
- **WISP compliance**: Written security plan required for all tax preparers
- **Segregation of duties**: Preparer and reviewer should be different people (when possible)

---

## Tax Return Lifecycle

### Workflow Stages

```
INTAKE → DOCUMENT COLLECTION → PREPARATION → REVIEW → APPROVAL → E-FILE → COMPLETE
   │              │                 │           │          │          │         │
   │              │                 │           │          │          │         │
   ▼              ▼                 ▼           ▼          ▼          ▼         ▼
Engagement    W-2/1099s         Data entry   QC check   Form 8879   Transmit  Archive
Client info   Upload/scan       Calculations  Errors    Signature   Wait ACK   7 years
Form 7216     OCR extract       Optimization  Review    Payment     Handle     Delivery
```

### Stage Definitions

| Stage | Entry Criteria | Exit Criteria | Key Actions |
|-------|----------------|---------------|-------------|
| **Intake** | New client contact | Engagement signed, 7216 signed | Collect contact info, sign engagement, collect 7216 consent |
| **Document Collection** | Intake complete | All required docs received | Upload W-2s, 1099s, prior return, ID verification |
| **Preparation** | Docs complete | Return calculated, no errors | Enter data, optimize, review calculations |
| **Review** | Prep complete | QC passed, reviewer approved | Technical review, check for errors/omissions |
| **Approval** | Review complete | 8879 signed, payment received | Client reviews return, signs 8879, pays fees |
| **E-File** | Approval complete | IRS accepted | Transmit to IRS, handle rejections |
| **Complete** | E-file accepted | Return archived | Deliver to client, archive documents |

---

## Return Types Overview

**Individual Returns:** 1040, 1040-SR, 1040-NR, 1040-X

**Business Returns:** 1120 (C-Corp), 1120-S (S-Corp), 1065 (Partnership), 1041 (Trust), 990 (Tax-Exempt)

**State Returns:** Due dates vary by state (usually match federal or shortly after)

For comprehensive return types table with due dates, extension forms, and document type details, consult **references/return-types-and-forms.md** covering:
- Individual return types and deadlines
- Business return types and deadlines
- State return requirements
- Income document types (W-2, all 1099 variants, K-1s)
- Deduction document types (1098s, receipts)
- Extension filing procedures (Form 4868, 7004)
- Estimated tax payment schedule (1040-ES)
- Annual tax calendar with all key dates

---

## IRS Circular 230 Summary

### Preparer Duties (Key Points)

- **Due diligence**: Must make reasonable inquiries if information appears incorrect
- **No frivolous positions**: Cannot take positions without reasonable basis
- **Written advice standards**: Written advice must consider all relevant facts
- **Client communication**: Must inform clients of penalties and return positions
- **Conflicts of interest**: Must disclose conflicts, obtain informed consent

### Prohibited Conduct

- Charging unconscionable fees
- Assisting in tax evasion
- Misappropriating client funds
- Practicing while suspended
- Giving false opinions

**Sanctions:** Private reprimand (minor) → Public censure (moderate) → Suspension (serious) → Disbarment (severe)

---

## WISP Requirements

### Written Information Security Plan (IRS Pub 4557)

Every tax preparer must have a written plan covering:

**Required Elements:**

1. **Designate Security Coordinator**
   - Named individual responsible for security
   - Contact information documented

2. **Risk Assessment**
   - Identify where client data is stored
   - Assess risks to data security
   - Document findings annually

3. **Safeguards**
   - Physical (locked files, secure office)
   - Technical (encryption, firewalls, MFA)
   - Administrative (policies, training)

4. **Employee Management**
   - Background checks for new hires
   - Security training (annual minimum)
   - Access controls (need-to-know basis)

5. **Incident Response Plan**
   - How to detect breaches
   - Notification procedures
   - Recovery steps

6. **Service Provider Oversight**
   - Vet third-party services
   - Contractual security requirements

**Annual Checklist:** Review/update WISP, conduct training, test incident response, update risk assessment, review service providers

---

## PTIN and ERO Requirements

**PTIN (Preparer Tax Identification Number):**
- Required for every paid preparer
- Annual renewal (~$30.75)
- Must appear on every return

**ERO (Electronic Return Originator):**
- Required to e-file returns
- EFIN (Electronic Filing Identification Number)
- IRS background check required
- Annual renewal

For detailed PTIN/ERO renewal procedures, compliance tracking, and requirements, consult **references/ptin-ero.md**.

---

## Form 7216 Consent

### Disclosure Consent Requirements

**When Required:**
- Before disclosing tax return information to ANY third party
- Before using tax return information for purposes other than tax prep
- Before sharing with cloud services, software providers (covered by blanket consent)

**Form Requirements:**
- Separate from engagement letter
- Cannot be pre-checked or assumed
- Must describe what will be disclosed and to whom
- Client signature required
- Must be retained with client file

**Valid Duration:**
- Typically one year
- Must be renewed for ongoing disclosures

**Exceptions (No 7216 Required):**
- Disclosure to IRS
- Disclosure required by court order
- Disclosure to other tax return preparers within same firm
- Quality or peer review

---

## Form 8879 (E-File Authorization)

### IRS e-file Signature Authorization

**Required Before:**
- Transmitting any e-filed return
- Cannot transmit without signed 8879

**Signature Methods:**
- Handwritten signature (wet ink)
- Electronic signature (if requirements met)
- PIN-based (for some e-file systems)

**Retention:**
- Must retain for 3 years from return due date
- Can be stored electronically if requirements met

**Electronic Signature Requirements:**
- Identity verification required before signing
- Must use electronic signature platform that captures:
  - Date and time of signature
  - IP address or device ID
  - Method of identity verification used

---

## Identity Verification

Tax preparers must verify client identity before filing.

**Verification Levels:**
- **Basic**: Prior year AGI match (returning clients)
- **Document**: Government ID + SSN card (new clients)
- **Enhanced**: Video call + document verification (high-risk, remote clients)

**Multi-Tier System:**
- **TIER_0**: Unverified (initial state)
- **TIER_1**: Returning Client (Name + SSN-4 + DOB + Prior Year AGI)
- **TIER_2**: New Client (Email + Phone + Gov ID + Persona Check)
- **TIER_3**: High-Risk (Enhanced Persona + Manual Review)

For detailed identity verification tier requirements, Persona integration specifications, phone verification limits, and verification component tracking, consult **references/identity-verification.md**.

---

## E-Filing Process

**Track acceptance/rejection for every transmission.**

**Acknowledgement Types:**
- **Accepted**: IRS received and accepted (return is filed)
- **Rejected**: IRS rejected - errors found (fix and retransmit)
- **Pending**: Still processing (wait, check again later)

**Common Rejections:**
- SSN already used (IND-031) → Verify SSN, may need paper file
- AGI mismatch (IND-181) → Correct prior year AGI or use IP PIN
- Name/SSN mismatch (IND-507) → Verify with SSA, correct spelling

For comprehensive rejection code table, rejection handling workflow code, e-filing architecture (3-layer structure), transmission status flow, rejection categories, escalation timelines, and auto-correctable errors, consult **references/e-filing.md**.

---

## Data Retention

**IRS requires 7-year retention for:**
- Tax returns (copy)
- Client workpapers
- Engagement letters
- ID verification docs

**Shorter retention:**
- Form 8879: 3 years from due date
- Form 7216: Duration of consent + 3 years

**Secure disposal after retention period:**
- Shred physical documents (cross-cut)
- Secure delete digital files
- Document disposal date

For detailed retention requirements table, secure disposal procedures, and retention schedule automation, consult **references/data-retention.md**.

---

## Workflow State Machine

The tax return workflow uses a 17-state machine with defined transitions:

**Key States:**
- INTAKE → DOCUMENTS_PENDING → READY_FOR_PREP → AI_ANALYSIS → IN_PREP
- IN_PREP → READY_FOR_REVIEW → IN_REVIEW → APPROVED → PENDING_SIGNATURE
- PENDING_SIGNATURE → READY_TO_FILE → FILED → ACCEPTED → COMPLETE

**Priority Levels:**
- Normal (standard turnaround, $0 rush fee)
- Rush 48 (48 hours, $150 rush fee)
- Rush 24 (24 hours, $300 rush fee)

For detailed 17-state workflow diagram, state progress percentages (5% → 100%), valid state transitions, priority definitions, rush fees, and workflow exception handling, consult **references/workflow-implementation.md**.

---

## Document Processing

**Pipeline:** UPLOADING → UPLOADED → SCANNING → CLASSIFYING → EXTRACTING → PROCESSING → VERIFIED → ARCHIVED

**Confidence Thresholds:**
- Classification: Auto-approve ≥90%, Review 70-89%, Reject <70%
- Extraction: Auto-approve ≥85%, Review 60-84%, Reject <60%

For detailed pipeline stage diagram, all confidence thresholds, status override procedures (MTG-006), and audit trail requirements, consult **references/document-processing.md**.

---

## Integration Partners

**SmartVault:** Client-facing document upload portal (bi-directional sync)

**SurePrep:** Primary OCR extraction for tax documents (W-2, 1099, K-1 field mapping)

**UltraTax:** Production tax preparation software (integrated e-filing)

**Google Workspace:** E-signatures (8879, engagement letters), client communications

**Stripe:** Payment processing (payment_intent_id, checkout_session_id tracking)

**Persona:** Enhanced identity verification (live document verification, facial match, confidence scoring)

For detailed integration specifications, field mappings, API patterns, and integration points, consult **references/integrations.md**.

---

## AI Integration

**AI-Powered Operations:**
- Classification: Document type identification (Claude)
- Extraction: Field extraction from documents (Claude)
- Analysis: Prior year comparison, missing docs (Claude)
- Q&A: Client questions about return (Claude)

**Batch Processing:** Schedule jobs for 2 AM (off-peak pricing), track token usage and costs

**Prior Year Variance:**
- Low (<10%): Informational only
- Medium (10-30%): Flag for preparer review
- High (>30%): Require explanation before filing

For detailed AI integration patterns, batch job structure, prompt execution tracking, variance detection code, and life event auto-detection, consult **references/ai-analysis.md**.

---

## Billing and Delivery

**Delivery Package Components (in order):**
1. Cover letter
2. Tax return
3. Form 8879 (signature required)
4. Invoice
5. Estimated tax vouchers

**Signature Workflow:** Package generated → Sent → Viewed → Signed → Ready for filing (reminders at days 3, 5, 7)

**Fee Calculation:** Base + (Base × Complexity) + Rush - Discount

**Complexity Multipliers:**
- Simple: 1.0x (standard 1040)
- Moderate: 1.25x (multiple schedules)
- Complex: 1.5x (investments, rentals)
- Very Complex: 2.0x (multi-state, K-1s, planning)

For detailed fee calculation code, base fee tables, aging buckets, reminder/escalation rules, and signature workflow constants, consult **references/billing-and-delivery.md**.

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Filing without 8879 | IRS penalty, invalid return | Always get signed 8879 first |
| Skipping ID verification | IRS compliance violation | Verify every client's identity |
| Missing 7216 consent | Potential $5,000 fine per violation | Get consent before ANY disclosure |
| No WISP documented | IRS requirement violation | Maintain written security plan |
| Expired PTIN | Cannot legally prepare returns | Renew annually before expiration |
| Ignoring rejections | Return not filed, penalties accrue | Handle rejections within 5 days |
| Discarding records early | Audit liability | Retain minimum 7 years |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Domain entities | `tax-practice/src/domain/` |
| Client model | `tax-practice/src/domain/client.py` |
| Return & workflow | `tax-practice/src/domain/workflow.py` |
| Document processing | `tax-practice/src/domain/document.py` |
| Extraction | `tax-practice/src/domain/extraction.py` |
| Analysis | `tax-practice/src/domain/analysis.py` |
| Review | `tax-practice/src/domain/review.py` |
| Engagement | `tax-practice/src/domain/engagement.py` |
| Delivery | `tax-practice/src/domain/delivery.py` |
| Invoice | `tax-practice/src/domain/invoice.py` |
| E-filing | `tax-practice/src/domain/efiling.py` |
| Identity verification | `tax-practice/src/domain/identity_verification.py` |
| Estimated tax | `tax-practice/src/domain/estimated_tax.py` |
| Services | `tax-practice/src/services/` |
| Workflows | `tax-practice/src/workflows/` |
| API schemas | `tax-practice/src/api/schemas/` |

---

## Detailed References

For comprehensive implementation details, consult the references/ directory:

- **references/return-types-and-forms.md** - Return types tables, document types, extensions, estimated tax, annual calendar
- **references/identity-verification.md** - Identity verification tiers, Persona integration, phone verification limits
- **references/e-filing.md** - Rejection codes, handling workflows, e-filing architecture (TD-020)
- **references/workflow-implementation.md** - 17-state machine, progress tracking, priority levels, exceptions
- **references/document-processing.md** - Pipeline stages, confidence thresholds, status overrides
- **references/integrations.md** - SmartVault, SurePrep, UltraTax, Google, Stripe, Persona integration specs
- **references/ai-analysis.md** - AI operations, batch processing, variance analysis, life event detection
- **references/billing-and-delivery.md** - Fee calculation, delivery packages, signature workflow, reminders
- **references/data-retention.md** - IRS retention requirements, secure disposal procedures
- **references/ptin-ero.md** - PTIN/ERO requirements, renewal procedures, compliance tracking

---

## References

- [IRS Publication 4557](https://www.irs.gov/pub/irs-pdf/p4557.pdf) - Safeguarding Taxpayer Data
- [IRS Circular 230](https://www.irs.gov/pub/irs-utl/circular_230.pdf) - Regulations Governing Practice
- [Form 7216 Instructions](https://www.irs.gov/pub/irs-pdf/i7216.pdf) - Consent Requirements
- [Form 8879 Instructions](https://www.irs.gov/pub/irs-pdf/i8879.pdf) - E-file Signature Authorization
- [IRS MeF Status Codes](https://www.irs.gov/e-file-providers/ack-and-error-codes) - Acknowledgement Codes
- [PTIN Requirements](https://www.irs.gov/tax-professionals/ptin-requirements-for-tax-return-preparers)

---

**Document Version:** 3.0
**Last Updated:** 2026-02-16
**Maintained By:** Aliunde Tax Practice AI Team

**Change Log:**
- v3.0 (2026-02-16): Progressive disclosure refactoring - moved ~700 lines to references/ directory (72% reduction)
- v2.0 (2026-01-02): Added system implementation patterns
- v1.0 (2025-12-14): Initial IRS compliance documentation
