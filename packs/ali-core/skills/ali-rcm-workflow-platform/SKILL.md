---
name: ali-rcm-workflow-platform
description: |
  RCM workflow automation platform architecture and integration patterns. Use when:

  PLANNING: Designing RCM platforms, routing engines, clearinghouse integrations, or rules engines

  IMPLEMENTATION: Building worklist prioritization, payer portal automation, EDI flows, or AR data models

  GUIDANCE: Best practices for queue management, SNIP validation, RPA in healthcare, or SLA tracking

  REVIEW: Auditing platform architecture, integration patterns, routing logic, or ML/analytics implementations
---

# RCM Workflow Platform

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing RCM platform architecture
- Planning clearinghouse or payer portal integrations
- Evaluating rules engine approaches

**Implementation:**
- Building worklist prioritization algorithms
- Implementing routing rules or queue management
- Creating payer portal automation (RPA)
- Building AR data models

**Guidance/Best Practices:**
- Asking about RCM platform vendors
- Asking about integration patterns
- Asking about ML use cases in RCM

**Review/Validation:**
- Reviewing platform architecture
- Auditing routing logic or priority scoring
- Validating integration patterns

---

## Key Principles

- **Priority-driven work** - Multi-factor scoring, not FIFO
- **SLA-first tracking** - Every work item has a deadline
- **Automation layering** - Auto → RPA → light touch → full work
- **Payer-specific logic** - Each payer has different rules
- **Real-time visibility** - Dashboards for queue health and staff productivity
- **EDI integration** - 835 for remits, 837 for claims, 270/271 for eligibility

---

## Platform Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RCM PLATFORM ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SOURCE SYSTEMS          INTEGRATION           CORE PROCESSING              │
│  ┌─────────────┐        ┌─────────────┐       ┌─────────────┐              │
│  │ EHR/EMR     │───────▶│ EDI Engine  │──────▶│ Claims      │              │
│  │ PMS         │        │ API Gateway │       │ Processing  │              │
│  │ Scheduling  │        │ SFTP/Batch  │       │ Denials     │              │
│  └─────────────┘        └─────────────┘       │ AR Mgmt     │              │
│                                                └─────────────┘              │
│                                                      │                       │
│                                                      ▼                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     WORKFLOW & RULES ENGINE                            │  │
│  │  Routing Rules │ Queue Mgmt │ SLA Monitor │ Alerts │ Task Assignment  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                      │                       │
│                                                      ▼                       │
│  EXTERNAL CONNECTIVITY                          ANALYTICS                   │
│  ┌─────────────┐  ┌─────────────┐              ┌─────────────┐             │
│  │Clearinghouse│  │Payer Portals│              │ Dashboards  │             │
│  │ (SFTP/API)  │  │ (RPA/API)   │              │ ML Models   │             │
│  └─────────────┘  └─────────────┘              └─────────────┘             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Core Components

| Layer | Components | Purpose |
|-------|------------|---------|
| **Presentation** | Web dashboards, mobile apps, work queues | User interaction |
| **Application** | Business logic, workflow orchestration, rules engine | Processing |
| **Integration** | EDI engine, API gateway, message queuing | External connectivity |
| **Data** | Claims DB, AR warehouse, document storage | Persistence |

---

## Patterns We Use

### Denial Routing Engine

```yaml
routing_rules:
  # Clinical denials to clinical team
  clinical_denials:
    conditions:
      - field: carc_code
        operator: in
        values: [CO-50, CO-55, CO-56, CO-150, CO-167, CO-236]
    destination:
      queue: clinical_appeals
      team: clinical_review_team
    priority_modifier: 1.2
    sla_days: 5

  # High-dollar routing
  high_value:
    conditions:
      - field: denial_amount
        operator: greater_than
        value: 10000
    destination:
      queue: high_dollar_queue
      team: senior_analysts
    sla_days: 3
    escalation_hours: 24

  # Payer-specific (UHC 65-day deadline)
  uhc_urgent:
    conditions:
      - field: payer_id
        operator: equals
        value: "87726"
    destination:
      queue: commercial_queue_a
      team: uhc_specialists
    priority_modifier: 1.5
    appeal_deadline_days: 65

  # Technical auto-correction
  technical_auto:
    conditions:
      - field: carc_code
        operator: in
        values: [CO-16, CO-18]
      - field: denial_amount
        operator: less_than
        value: 100
    destination:
      queue: auto_review
      auto_process: true
```

### Multi-Factor Priority Scoring

```python
def calculate_priority(denial):
    """
    Composite priority score combining multiple factors.
    Higher score = work first.
    """
    weights = {
        'dollar': 0.30,      # Higher value = higher priority
        'urgency': 0.25,     # Days to deadline
        'overturn': 0.20,    # Historical success rate for this CARC/payer
        'age': 0.15,         # AR aging bucket
        'payer_history': 0.10
    }

    scores = {}

    # Dollar score (log scale to prevent extreme values dominating)
    scores['dollar'] = min(math.log10(max(denial.amount, 100)) / 5 * 100, 100)

    # Urgency score (exponential as deadline approaches)
    days_remaining = (denial.appeal_deadline - today()).days
    if days_remaining <= 0:
        scores['urgency'] = 100
    elif days_remaining <= 7:
        scores['urgency'] = 90 + (7 - days_remaining)
    elif days_remaining <= 14:
        scores['urgency'] = 70 + (14 - days_remaining) * 2
    else:
        scores['urgency'] = max(0, 40 - (days_remaining - 30))

    # Historical overturn rate for this CARC/payer combination
    scores['overturn'] = get_overturn_rate(denial.payer_id, denial.carc_code) * 100

    # Age score
    scores['age'] = min(denial.age_days / 120 * 100, 100)

    # Payer success rate
    scores['payer_history'] = get_payer_success_rate(denial.payer_id) * 100

    return sum(weights[k] * scores[k] for k in weights)
```

### Expected Value Optimization

```python
def optimize_worklist_ev(denials, staff_hours_available):
    """
    Maximize expected recovery value given limited staff time.
    Classic knapsack optimization.
    """
    for denial in denials:
        prob = predict_overturn_probability(denial)
        time_to_work = estimate_work_time(denial)
        denial.expected_value = denial.amount * prob
        denial.ev_per_hour = denial.expected_value / time_to_work

    # Sort by EV per hour (efficiency)
    denials.sort(key=lambda d: d.ev_per_hour, reverse=True)

    # Select items that fit within available hours
    selected = []
    total_hours = 0

    for denial in denials:
        if total_hours + denial.time_to_work <= staff_hours_available:
            selected.append(denial)
            total_hours += denial.time_to_work

    return selected
```

---

## Clearinghouse Integration

### Integration Methods

| Method | Use Case | Latency | Volume |
|--------|----------|---------|--------|
| **SFTP Batch** | Claims submission, remits | Hours | High |
| **Real-Time API** | Eligibility, claim status | Seconds | Per-transaction |
| **Webhooks** | Event notifications | Near real-time | Event-driven |

### SFTP Batch Configuration

```yaml
sftp_integration:
  connection:
    host: sftp.clearinghouse.com
    port: 22
    authentication: ssh_key

  outbound:
    claims_837:
      directory: /outbound/claims
      schedule: "*/15 * * * *"  # Every 15 minutes
      file_pattern: "837_{timestamp}_{batch_id}.edi"

  inbound:
    remits_835:
      directory: /inbound/remits
      poll_interval: 300  # 5 minutes
      file_pattern: "835_*.edi"
```

### Real-Time API Integration

```yaml
api_integration:
  base_url: https://api.clearinghouse.com/v2
  authentication:
    type: oauth2
    token_endpoint: /auth/token

  endpoints:
    eligibility:
      method: POST
      path: /eligibility/270
      timeout_ms: 30000

    claim_status:
      method: POST
      path: /claims/276
      async: true
      webhook_url: https://ourapp.com/webhooks/claim-status
```

### SNIP Validation Levels

| Level | Name | Validation |
|-------|------|------------|
| 1 | EDI Compliance | X12 syntax, segment structure |
| 2 | HIPAA Requirement | Required fields, valid codes |
| 3 | Balance | Header/detail counts match |
| 4 | Inter-Segment | Cross-segment relationships |
| 5 | External Code Sets | ICD, CPT, NPI validity |
| 6 | Payer-Specific | Payer business rules |
| 7 | Claim-Specific | Trading partner requirements |

---

## Payer Portal Automation (RPA)

### RPA Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     PAYER PORTAL AUTOMATION                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  RCM PLATFORM ───▶ RPA ORCHESTRATOR ───▶ PAYER PORTALS ───▶ DATA EXTRACT   │
│                          │                                       │           │
│                    ┌─────┴─────┐                                 ▼           │
│                    │  BOT FARM │                           ┌─────────┐       │
│                    │ ┌───┐┌───┐│                           │ UPDATE  │       │
│                    │ │Bot││Bot││                           │ AR/CLAIMS│      │
│                    │ └───┘└───┘│                           └─────────┘       │
│                    └───────────┘                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Common RPA Use Cases

| Use Case | Manual Time | Automated Time | Improvement |
|----------|-------------|----------------|-------------|
| Eligibility check | 21 min | <1 min | 95%+ |
| Claim status check | 18 min | <2 min | 89% |
| Prior auth submission | 45 min | 5 min | 89% |
| Denial follow-up | 25 min | 3 min | 88% |

### Eligibility Bot Workflow

```yaml
eligibility_bot:
  triggers:
    - schedule: "0 6 * * *"  # Daily 6 AM
    - event: new_appointment

  workflow:
    - step: login
      portal: "${payer_portal_url}"
      credentials: "${vault.payer_credentials}"

    - step: navigate
      path: /eligibility/inquiry

    - step: input
      fields:
        member_id: "${patient.member_id}"
        dob: "${patient.dob}"
        service_date: "${appointment.date}"

    - step: extract
      data:
        - coverage_status
        - effective_date
        - deductible
        - deductible_met
        - authorization_required

    - step: update
      target: patient_record
```

---

## Backend AR Data Model

### Core Tables

```sql
-- Claims table
CREATE TABLE claims (
    claim_id            UUID PRIMARY KEY,
    patient_id          UUID NOT NULL,
    payer_id            UUID NOT NULL,
    internal_claim_num  VARCHAR(50) NOT NULL,
    payer_claim_num     VARCHAR(50),

    -- Financial
    total_charges       DECIMAL(12,2) NOT NULL,
    total_paid          DECIMAL(12,2) DEFAULT 0,
    total_adjusted      DECIMAL(12,2) DEFAULT 0,
    balance             DECIMAL(12,2) GENERATED ALWAYS AS
                        (total_charges - total_paid - total_adjusted),

    -- Status
    claim_status        VARCHAR(20) NOT NULL,
    service_date_from   DATE NOT NULL,
    submission_date     TIMESTAMP,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_claims_balance ON claims(balance) WHERE balance > 0;
CREATE INDEX idx_claims_status ON claims(claim_status);

-- Denials table (linked to claims)
CREATE TABLE denials (
    denial_id           UUID PRIMARY KEY,
    claim_id            UUID REFERENCES claims(claim_id),
    denial_amount       DECIMAL(12,2) NOT NULL,

    -- CARC/RARC
    group_code          VARCHAR(2) NOT NULL,
    carc_code           VARCHAR(10) NOT NULL,
    rarc_codes          VARCHAR(10)[],

    -- Workflow
    workflow_status     VARCHAR(30) DEFAULT 'new',
    assigned_queue      VARCHAR(50),
    priority_score      DECIMAL(5,2),
    appeal_deadline     DATE,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_denials_queue ON denials(assigned_queue, workflow_status);
CREATE INDEX idx_denials_deadline ON denials(appeal_deadline);

-- Work queue items
CREATE TABLE work_queue_items (
    item_id             UUID PRIMARY KEY,
    item_type           VARCHAR(30) NOT NULL,  -- denial, claim, appeal
    reference_id        UUID NOT NULL,
    queue_id            UUID NOT NULL,
    assigned_user       UUID,

    priority_score      DECIMAL(5,2) NOT NULL,
    sla_deadline        TIMESTAMP,
    sla_status          VARCHAR(20),  -- on_track, at_risk, breached

    status              VARCHAR(20) DEFAULT 'pending',
    touch_count         INTEGER DEFAULT 0,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_queue_priority ON work_queue_items(priority_score DESC);
```

### AR Aging View

```sql
CREATE VIEW ar_aging_summary AS
SELECT
    payer_id,
    payer_name,
    SUM(CASE WHEN days_outstanding <= 30 THEN balance END) as current_0_30,
    SUM(CASE WHEN days_outstanding BETWEEN 31 AND 60 THEN balance END) as aged_31_60,
    SUM(CASE WHEN days_outstanding BETWEEN 61 AND 90 THEN balance END) as aged_61_90,
    SUM(CASE WHEN days_outstanding BETWEEN 91 AND 120 THEN balance END) as aged_91_120,
    SUM(CASE WHEN days_outstanding > 120 THEN balance END) as aged_120_plus,
    SUM(balance) as total_ar
FROM (
    SELECT c.*, p.payer_name,
           CURRENT_DATE - c.service_date_from as days_outstanding
    FROM claims c
    JOIN payers p ON c.payer_id = p.payer_id
    WHERE c.balance > 0
) aged_claims
GROUP BY payer_id, payer_name;
```

---

## Rules Engine Design

### JSON Rule Format

```json
{
  "rule_id": "RULE-001",
  "name": "Route Medical Necessity Denials",
  "priority": 100,
  "enabled": true,

  "conditions": {
    "all": [
      {
        "fact": "denial.carc_code",
        "operator": "in",
        "value": ["CO-50", "CO-55", "CO-56"]
      },
      {
        "fact": "denial.amount",
        "operator": "greaterThan",
        "value": 500
      }
    ]
  },

  "actions": [
    {
      "type": "route",
      "params": {
        "queue": "clinical_review",
        "team": "clinical_appeals_team"
      }
    },
    {
      "type": "set_sla",
      "params": { "days": 5 }
    }
  ]
}
```

### Decision Table Format

```yaml
decision_table:
  name: Denial Routing Matrix
  hit_policy: first  # first, all, collect

  inputs:
    - name: carc_category
    - name: amount_tier
    - name: payer_type

  outputs:
    - name: queue
    - name: priority
    - name: sla_days

  rules:
    - inputs: [clinical, high, "*"]
      outputs: [senior_clinical, critical, 3]

    - inputs: [clinical, medium, "*"]
      outputs: [clinical_review, high, 5]

    - inputs: [authorization, "*", uhc]
      outputs: [uhc_auth_team, high, 3]

    - inputs: [technical, "*", "*"]
      outputs: [corrections_queue, normal, 3]
```

---

## ML/Analytics in RCM

### Denial Prediction Features

| Feature Category | Examples |
|------------------|----------|
| **Claim-level** | Total charges, service type, place of service, diagnosis codes |
| **Patient-level** | Prior denials, payer history, demographics |
| **Provider-level** | Historical denial rate, specialty, NPI |
| **Payer-level** | Payer denial rate, policy changes, prior auth requirements |
| **Temporal** | Day of week submitted, month, holiday proximity |

### Appeal Success Prediction

```python
def predict_appeal_success(denial):
    features = {
        'carc_code': denial.carc_code,
        'payer_id': denial.payer_id,
        'denial_amount': denial.amount,
        'has_clinical_docs': denial.has_clinical_documentation,
        'provider_specialty': denial.provider_specialty,
        'days_since_service': denial.days_since_service,
        'prior_appeals_this_claim': denial.prior_appeal_count,
        'payer_historical_overturn': get_payer_overturn_rate(denial.payer_id)
    }

    probability = appeal_model.predict_proba(features)[1]

    return {
        'probability': probability,
        'recommendation': 'appeal' if probability > 0.40 else 'review',
        'expected_value': denial.amount * probability
    }
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Static priority (not recalculating) | Stale rankings, missed deadlines | Recalculate daily or on status change |
| FIFO work order | Ignores value and probability | Multi-factor priority scoring |
| Missing SLA tracking | Breached deadlines unknown | Track SLA status, escalate at-risk |
| Unmanaged RPA credentials | Security risk | Vault integration, rotation |
| No fallback for RPA failures | Silent failures | Alert + manual queue fallback |
| Hardcoded routing rules | Difficult to update | Rules engine with UI |
| Single-threaded bot execution | Slow throughput | Parallel bot farm |

---

## Quick Reference

### EDI Transaction Summary

| Transaction | Name | Direction | Purpose |
|-------------|------|-----------|---------|
| 270 | Eligibility Inquiry | Provider → Payer | Check coverage |
| 271 | Eligibility Response | Payer → Provider | Return coverage |
| 276 | Claim Status Inquiry | Provider → Payer | Check claim |
| 277 | Claim Status Response | Payer → Provider | Return status |
| 837P/I | Claim | Provider → Payer | Submit claim |
| 835 | Remittance | Payer → Provider | Payment/denial |

### Clearinghouse Comparison

| Vendor | Payer Connections | Key Features |
|--------|-------------------|--------------|
| **Availity** | 2,000+ | Real-time eligibility, multi-payer portal |
| **Change HC** | 5,000+ | EDI + analytics |
| **TriZetto** | 8,000+ | 98% auto-adjudication |
| **Waystar** | 5,000+ | AI-powered, Epic certified |

### RPA Platform Options

| Platform | Healthcare Focus | Integration |
|----------|------------------|-------------|
| UiPath | 75% of top 100 health systems | EHR connectors |
| Blue Prism | Enterprise-grade | HIPAA-compliant |
| Automation Anywhere | Cloud-native | Healthcare templates |
| Power Automate | Low-code | Azure health APIs |

---

## References

- [Availity Portal](https://www.availity.com/)
- [Waystar Platform](https://www.waystar.com/)
- [UiPath Healthcare](https://www.uipath.com/solutions/industry/healthcare)
- [X12 Transaction Standards](https://x12.org/)

---

## Related Skills

- **rcm-denial-management** - Denial classification, CARC-based routing rules, appeal workflows
- **rcm-low-balance-ar** - Low-value account handling, cost-to-collect economics
- **x12-edi-core** - EDI structure, ISA/GS/ST envelopes, delimiter handling
- **x12-835-payment** - Remittance parsing, CAS segments, denial extraction
- **x12-837i-institutional** - Claim submission, UB-04 mapping
- **ai-development** - ML model patterns for denial prediction and appeal scoring

---

**Document Version:** 1.1
**Last Updated:** 2026-01-20
**Source:** RCM Workflow Automation Platform Knowledge Base
