# ali-tax-practice-domain - Integration Partners Reference

This reference contains detailed integration specifications for all third-party services used in the tax practice system.

---

## Integration Partners

### SmartVault (Document Portal)

**Purpose:** Client-facing document upload portal

**Integration Points:**
- Client-facing document upload portal
- Bi-directional sync with internal document store
- Auto-import new documents to pending queue

**Key Features:**
- Secure client portal access
- Document organization by client/year
- Automatic notifications on upload
- Integration with internal document tracking

---

### SurePrep (OCR & UltraTax Bridge)

**Purpose:** Primary OCR extraction and tax software bridge

**Integration Points:**
- Primary OCR extraction for tax documents
- Bridge to UltraTax for tax preparation
- Handles W-2, 1099, K-1 extraction with field mapping

**Supported Documents:**
- W-2 (wages and withholding)
- All 1099 variants (INT, DIV, B, MISC, NEC, R, G, K)
- K-1 forms (1065, 1120-S, 1041)

**Field Mapping:**
- Automatic field extraction to UltraTax format
- Confidence scoring per field
- Manual override capability

---

### UltraTax (Tax Preparation)

**Purpose:** Production tax preparation software

**Features:**
- Handles calculations, optimization, forms
- Integrated e-filing (no separate API needed)
- Multi-state return support
- Amendment tracking

**Integration:**
- Receives data from SurePrep bridge
- Returns completed forms for review
- Handles e-filing transmission directly

---

### Google Workspace

**Purpose:** Document signatures and client communications

| Service | Purpose |
|---------|---------|
| Docs/Drive | E-signatures (8879, engagement letters) |
| Calendar | Estimated tax payment reminders |
| Gmail | Client communications |

**Signature Workflow:**
- Form 8879 via Google Docs signature
- Engagement letters via Google Docs
- Automated reminders via Calendar
- Client email communications via Gmail

---

### Stripe (Payments)

**Purpose:** Payment processing for preparation fees

**Integration Fields:**
- `payment_intent_id`: Stripe payment reference
- `checkout_session_id`: Stripe Checkout session
- Payment status tracking
- Refund handling

**Features:**
- Secure payment processing
- Automatic invoice generation
- Payment reminders
- Refund processing

---

### Persona (Identity Verification)

**Purpose:** Enhanced identity verification for remote clients

**Features:**
- Live document verification
- Facial match (selfie to ID)
- Confidence scoring with auto-approval
- Risk signal detection

**Integration Fields:**
- `persona_inquiry_id`: Unique inquiry reference
- `persona_confidence`: Confidence score (0.0-1.0)
- Verification component flags (selfie_match, document_authentic, etc.)

**Verification Components:**
- Government ID capture
- Selfie capture and facial match
- Liveness detection
- Document authenticity check
- Data consistency validation
- Expiration date verification
