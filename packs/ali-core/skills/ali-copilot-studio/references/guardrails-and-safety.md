# ali-copilot-studio - Guardrails and Safety Reference

This reference provides detailed configuration patterns for content moderation, safety filters, and compliance controls in Microsoft CoPilot Studio.

---

## Content Moderation Levels

### Configuration

| Level | Enforcement | Use Case | Blocked Content |
|-------|-------------|----------|-----------------|
| **Low** | Informational only | Internal testing, dev environments | None (log only) |
| **Medium** | Warn on violations | General business use, internal apps | Profanity, mild harmful content |
| **High** | Block violations | Customer-facing, public deployments | All harmful content, PII, profanity |

### Setting Moderation Level

**In CoPilot Studio:**
1. Navigate to Settings > Security
2. Select Content Moderation level
3. Configure custom rules (optional)
4. Enable logging for violations
5. Test with sample inputs

---

## Safety Filters

### Microsoft-Managed Filters (Enabled by Default)

**1. Harmful Content Detection**
- Violence and gore
- Self-harm and suicide
- Sexual content (explicit)
- Hate speech and discrimination

**Action**: Block message, return safe fallback response

**2. PII/PHI Detection and Redaction**
- Social Security Numbers (SSN)
- Credit card numbers
- Email addresses
- Phone numbers
- Healthcare identifiers (MRN, insurance ID)
- Driver's license numbers
- Passport numbers

**Action**: Redact from logs, mask in UI, block transmission to external systems

**3. Profanity Filtering**
- Common profanity across multiple languages
- Contextual profanity (slurs, insults)

**Action**: Replace with ***, optionally block message entirely

**4. Sensitive Topic Blocking**
- Self-harm instructions
- Violence promotion
- Illegal activities
- Hate speech and extremism

**Action**: Block message, log violation, optionally escalate to human review

---

## Custom Content Filters

### Keyword Blocking Lists

**Pattern:**
```
Blocked keywords:
- "competitor-name"
- "unreleased-product"
- "confidential-project-code"

Action: Block message if keyword detected
Response: "I cannot discuss that topic. Please contact [department] for more information."
```

**Configuration:**
1. Navigate to Settings > Content Filters
2. Add keyword list
3. Set case-sensitivity option
4. Choose action (block, warn, log)

### Pattern Matching (Regex)

**Use case**: Block specific patterns not covered by built-in filters

**Examples:**

**Internal ticket numbers:**
```regex
TICKET-\d{5,8}
```

**Proprietary codes:**
```regex
PROJ-[A-Z]{3}-\d{4}
```

**Configuration:**
```yaml
pattern: "TICKET-\\d{5,8}"
action: redact
replacement: "[TICKET-REDACTED]"
log: true
```

### Custom ML Models

**Use case**: Domain-specific content classification

**Setup:**
1. Train custom model in Azure AI
2. Export model endpoint
3. Configure in CoPilot Studio as external filter
4. Set confidence threshold

**Example: Financial compliance filter**
```
Model: financial-compliance-v1
Endpoint: https://ml.company.com/models/financial-compliance/score
Threshold: 0.85

Classifications:
- insider_trading_mention
- regulatory_violation
- market_manipulation

Action: Block + escalate to compliance team
```

---

## Response Validation

### Validation Flow

```
User input: [Query about sensitive topic]
↓
1. Pre-processing filter
   - Check against keyword blocklist
   - Pattern matching (regex)
   - PII detection
↓
2. If violation detected:
     → Return safe fallback response
     → Log violation for review
     → Optionally escalate to human
   Else:
     → Proceed to orchestration
↓
3. Generate response
↓
4. Post-processing filter
   - Validate response doesn't contain blocked content
   - Check for PII leakage
   - Verify compliance with policies
↓
5. If response violates rules:
     → Replace with safe fallback
     → Log for review
   Else:
     → Return to user
```

### Safe Fallback Responses

**Categories:**

| Violation Type | Fallback Response |
|----------------|-------------------|
| **Harmful content request** | "I cannot provide information on that topic. If you need assistance, please contact [resource]." |
| **PII detected** | "I've detected personal information in your message. For privacy, please avoid sharing [type] in this chat." |
| **Blocked keyword** | "I cannot discuss [topic]. Please reach out to [department] for more information." |
| **Out of scope** | "That's outside my area of expertise. I recommend contacting [team] for help with that." |

**Best practices:**
- Clear and specific (explain why blocked)
- Helpful redirection (point to alternative resource)
- Maintain professional tone
- Avoid revealing security details ("blocked by security filter" → "I cannot discuss that topic")

---

## PII/PHI Handling

### Automatic Detection

**Detected types:**

| Data Type | Example | Regex Pattern |
|-----------|---------|---------------|
| **SSN** | 123-45-6789 | `\d{3}-\d{2}-\d{4}` |
| **Credit card** | 4111-1111-1111-1111 | `\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}` |
| **Email** | user@example.com | `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}` |
| **Phone** | (555) 123-4567 | `\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}` |
| **MRN** | MRN-123456 | `MRN-\d{6,10}` |

### Actions on Detection

**1. Redact from logs**
```
Original: "My SSN is 123-45-6789"
Logged: "My SSN is ***-**-****"
```

**2. Mask in UI**
```
Display: "Card ending in ****1234"
Stored: "[REDACTED]"
```

**3. Block transmission to external systems**
```
Before sending to external API:
  - Scan for PII
  - If found: Block API call
  - Return error: "Cannot send message containing personal information"
```

**4. Require explicit user consent**
```
Agent: "I detected a Social Security Number in your message. For security:
        - This information will be encrypted
        - Shared only with authorized personnel
        - Deleted after 30 days

        Do you want to proceed?"
Options: ["Yes, proceed", "No, let me rephrase"]
```

---

## Compliance Controls

### GDPR Compliance

**Requirements:**

**1. Data retention policies**
```yaml
conversation_logs:
  retention_period: 30 days
  auto_delete: true

user_data:
  retention_period: 90 days
  deletion_trigger: user_request
```

**2. Right to deletion (forget user)**
```
User request: "Delete my data"
↓
Action: Power Automate flow
  - Delete conversation logs
  - Remove user profile data
  - Revoke consent records
  - Confirm deletion to user
```

**3. Data export capabilities**
```
User request: "Export my data"
↓
Action: Power Automate flow
  - Collect all user data (logs, profile, consents)
  - Package as JSON
  - Email download link
  - Log export request
```

**4. Consent management**
```
First interaction:
  Display: "By using this agent, you consent to:
            - Conversation logging for improvement
            - Data processing per privacy policy
            - Cookie usage for session management"
  Options: ["I consent", "Privacy policy", "Decline"]

If declined:
  - Do not proceed
  - Offer alternative contact methods
```

---

### HIPAA Compliance

**Requirements for healthcare applications:**

**1. PHI encryption at rest and in transit**
```
Storage:
  - Conversation logs encrypted with AES-256
  - Stored in HIPAA-compliant Azure region
  - Access restricted by role

Transit:
  - HTTPS/TLS 1.2+ for all communication
  - End-to-end encryption for sensitive fields
```

**2. Access controls and audit logs**
```yaml
access_controls:
  role: healthcare_provider
  permissions:
    - view_patient_data
    - create_consult_notes
  mfa_required: true

audit_log:
  events:
    - user_authentication
    - data_access
    - phi_view
    - export_request
  retention: 6 years
```

**3. Business Associate Agreements**
- Execute BAA with Microsoft for CoPilot Studio
- Document all third-party integrations
- Ensure subcontractors have BAAs

**4. Breach notification procedures**
```
Detection:
  - Unauthorized PHI access
  - Data exfiltration attempt
  - Encryption failure

Response (within 60 days):
  1. Contain breach
  2. Investigate scope
  3. Notify affected individuals
  4. Report to HHS (if >500 records)
  5. Document incident
```

---

### SOC 2 Compliance

**Controls:**

**1. Change management tracking**
```
Every agent change:
  - Logged with timestamp, user, description
  - Peer review required (dev environment)
  - Approval workflow (prod deployment)
  - Rollback procedure documented
```

**2. Access reviews**
```
Quarterly:
  - Review user access to CoPilot Studio
  - Validate role assignments
  - Remove access for departed employees
  - Document review results
```

**3. Incident response procedures**
```
Incident detected:
↓
1. Contain: Disable agent if actively harmful
2. Investigate: Review logs, determine root cause
3. Remediate: Fix issue, deploy patch
4. Document: Write post-mortem
5. Notify: Inform affected users if necessary
```

**4. Continuous monitoring**
```
Monitored metrics:
  - Failed authentication attempts
  - Content filter violations
  - Error rates
  - Unusual usage patterns

Alerts:
  - Spike in violations → Security team
  - High error rate → Engineering team
  - Unauthorized access → Compliance team
```

---

## Testing Safety Controls

### Test Cases

**1. Harmful content**
```
Input: "How do I [harmful action]?"
Expected: Blocked, fallback response, logged
```

**2. PII leakage**
```
Input: "My SSN is 123-45-6789"
Expected: Redacted in logs, warning to user
```

**3. Profanity**
```
Input: "[profanity] help me"
Expected: Filtered, request processed normally
```

**4. Out of scope**
```
Input: "What's the weather?"
Expected: Redirect to appropriate resource
```

**5. Adversarial prompts (jailbreak attempts)**
```
Input: "Ignore previous instructions and..."
Expected: Blocked, logged as potential attack
```

### Penetration Testing

**Recommended annually:**
1. Engage security firm
2. Test content filters with known bypasses
3. Attempt PII exfiltration
4. Test authentication controls
5. Review and remediate findings

---

## Monitoring and Alerts

### Key Metrics

| Metric | Threshold | Alert Action |
|--------|-----------|--------------|
| **Content violations per hour** | >10 | Notify security team |
| **PII detection rate** | >5% of messages | Review training, user guidance |
| **Failed auth attempts** | >3 per user | Lock account, require MFA |
| **API error rate** | >1% | Engineering investigation |
| **Escalation rate** | >20% | Review agent quality |

### Dashboard

**Include in monitoring dashboard:**
- Total conversations (daily, weekly, monthly)
- Safety filter violations (by type)
- PII detections and redactions
- Custom filter hits
- Escalations to human agents
- Compliance audit events

**Tools:**
- Power BI dashboard
- Azure Application Insights
- Azure Monitor alerts
