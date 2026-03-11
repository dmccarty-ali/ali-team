---
name: ali-rcm-denial-management
description: |
  Healthcare Revenue Cycle denial management expertise. Use when:

  PLANNING: Designing denial workflows, queue routing logic, appeal processes, or root cause analysis

  IMPLEMENTATION: Building denial categorization, CARC/RARC parsing, appeal tracking, or KPI dashboards

  GUIDANCE: Best practices for denial prevention, overturn rates, payer-specific appeal requirements

  REVIEW: Auditing denial management processes, KPIs, compliance with appeal deadlines
---

# RCM Denial Management

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing denial management workflows or queue routing
- Planning appeal processes or escalation paths
- Evaluating denial prevention strategies

**Implementation:**
- Building denial categorization from CARC/RARC codes
- Implementing appeal deadline tracking
- Creating denial routing rules or priority scoring

**Guidance/Best Practices:**
- Asking about payer-specific appeal requirements
- Asking about denial KPI benchmarks
- Asking about root cause analysis methodology

**Review/Validation:**
- Reviewing denial management processes
- Auditing appeal deadline compliance
- Validating denial routing logic

---

## Key Principles

- **Prevention over cure** - 86% of denials are preventable; invest in front-end processes
- **Deadline-first prioritization** - Appeal deadlines are non-negotiable; track by payer
- **Root cause focus** - Fix upstream issues, don't just work individual denials
- **Clinical vs technical triage** - Route to appropriate expertise based on denial type
- **Data-driven decisions** - Use CARC/RARC codes to drive categorization, not manual review
- **Payer-specific workflows** - Each payer has different rules, deadlines, and appeal paths

---

## Denial Classification Framework

### Clinical vs Technical Denials

| Type | % of Denials | Characteristics | Resolution |
|------|--------------|-----------------|------------|
| **Clinical (Hard)** | ~31% | Medical necessity, level of care, clinical validation | Formal appeal with documentation |
| **Technical (Soft)** | ~27% | Missing data, eligibility, authorization, duplicate | Correct and resubmit |

### Clinical Denial Types

| Type | CARC Codes | Root Cause | Resolution |
|------|------------|------------|------------|
| Medical Necessity | CO-50, CO-56 | Insufficient clinical documentation | CDI improvement, peer-to-peer |
| Level of Care | CO-55, CO-150 | Wrong care setting (IP vs OP) | Two-midnight documentation |
| Clinical Validation | CO-167 | Diagnosis doesn't support treatment | Coding/documentation alignment |
| Experimental | CO-236 | Service not proven effective | Clinical literature appeal |

### Technical Denial Types

| Type | CARC Codes | Root Cause | Resolution |
|------|------------|------------|------------|
| Eligibility | CO-22, CO-26, CO-109 | Coverage verification failure | Verify and resubmit |
| Authorization | CO-197, CO-15 | Missing prior auth | Retroactive auth if possible |
| Timely Filing | CO-29 | Submission deadline missed | Appeal with proof of timely filing |
| Duplicate | CO-18 | Same service billed twice | Verify, add modifier if distinct |
| Missing Data | CO-16 | Required fields missing | Correct and resubmit |
| Bundling | CO-97 | Service included in another | Review NCCI edits, appeal if distinct |

---

## Patterns We Use

### CARC-Based Routing Rules

```yaml
routing_rules:
  clinical_denials:
    conditions:
      - carc_code IN [CO-50, CO-55, CO-56, CO-150, CO-167, CO-236]
    destination:
      queue: clinical_appeals
      team: clinical_review_team
    sla_days: 5

  authorization_denials:
    conditions:
      - carc_code IN [CO-197, CO-15, CO-27]
    destination:
      queue: authorization_queue
      team: auth_specialists
    sla_days: 3

  coding_denials:
    conditions:
      - carc_code IN [CO-4, CO-11, CO-97]
    destination:
      queue: coding_corrections
      team: certified_coders
    sla_days: 3

  technical_denials:
    conditions:
      - carc_code IN [CO-16, CO-18, CO-22]
    destination:
      queue: technical_queue
      auto_correct: true
    sla_days: 2
```

### Priority Scoring Algorithm

```python
def calculate_priority(denial):
    weights = {
        'dollar': 0.30,      # Higher value = higher priority
        'urgency': 0.25,     # Days to deadline
        'overturn': 0.20,    # Historical success rate
        'age': 0.15,         # AR aging bucket
        'payer_history': 0.10
    }

    scores = {
        'dollar': log_scale(denial.amount, min=100, max=100000),
        'urgency': urgency_score(denial.appeal_deadline),
        'overturn': payer_overturn_rate(denial.payer_id, denial.carc),
        'age': age_bucket_score(denial.service_date),
        'payer_history': payer_success_rate(denial.payer_id)
    }

    return sum(weights[k] * scores[k] for k in weights)
```

### Multi-Level Appeal Tracking

```sql
CREATE TABLE appeals (
    appeal_id           UUID PRIMARY KEY,
    denial_id           UUID REFERENCES denials(denial_id),
    appeal_level        INTEGER NOT NULL DEFAULT 1,
    appeal_type         VARCHAR(30),  -- standard, expedited, peer_to_peer
    submitted_date      DATE,
    expected_response   DATE,
    response_status     VARCHAR(30),  -- pending, approved, denied, partial
    response_amount     DECIMAL(12,2)
);

-- Track escalation path
SELECT
    d.denial_id,
    d.carc_code,
    d.denial_amount,
    a1.response_status as level_1_status,
    a2.response_status as level_2_status,
    a3.response_status as level_3_status
FROM denials d
LEFT JOIN appeals a1 ON d.denial_id = a1.denial_id AND a1.appeal_level = 1
LEFT JOIN appeals a2 ON d.denial_id = a2.denial_id AND a2.appeal_level = 2
LEFT JOIN appeals a3 ON d.denial_id = a3.denial_id AND a3.appeal_level = 3;
```

---

## Payer-Specific Appeal Requirements

### Appeal Timeline Summary

| Payer | Commercial Window | MA Window | Response Time | Notes |
|-------|-------------------|-----------|---------------|-------|
| **UnitedHealthcare** | 65 days | 60 days | 30-60 days | Shortest commercial; digital required 2025+ |
| **Aetna** | 120-180 days | 60 days | 30-60 days | Plan-dependent |
| **Cigna** | 180 days | 60 days | 30-60 days | P2P recommended first |
| **BCBS** | 180 days (varies) | 60 days | 30-60 days | Check state-specific rules |
| **Medicare FFS** | 120 days (L1) | N/A | 60 days | 5-level appeal process |
| **Medicaid** | State-specific | N/A | Varies | Typically 60 days |

### Medicare Appeal Levels

| Level | Name | Filing Deadline | Decision Deadline |
|-------|------|-----------------|-------------------|
| 1 | Redetermination | 120 days | 60 days |
| 2 | Reconsideration (QIC) | 180 days | 60 days |
| 3 | ALJ/OMHA Hearing | 60 days | 90 days |
| 4 | Medicare Appeals Council | 60 days | 90 days |
| 5 | Federal District Court | 60 days | N/A |

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Missing appeal deadlines | Revenue permanently lost | Track deadlines in system, escalate at 10 days remaining |
| Routing all denials to one queue | Backlogs, expertise mismatch | CARC-based routing to specialized teams |
| Ignoring pattern analysis | Same errors repeat | Weekly root cause review, fix upstream |
| Manual denial categorization | Inconsistent, slow | Auto-categorize from CARC/RARC codes |
| Working denials by age only | Miss high-value opportunities | Multi-factor priority scoring |
| Over-reliance on write-offs | Revenue leakage | Write-off only after appeal exhausted |
| Ignoring P2P option | Miss fastest resolution path | Request P2P for clinical denials immediately |

---

## Key Performance Indicators

### Primary KPIs

| Metric | Formula | Benchmark | Target |
|--------|---------|-----------|--------|
| **Initial Denial Rate** | Denied $ / Total $ | 10-12% | <5% |
| **Overturn Rate** | Overturned / Appealed | 50% | >60% |
| **Appeal Rate** | Appealed / Denied | 63% | >80% |
| **First Pass Rate** | Paid First Submission / Total | 85% | >95% |

### Secondary KPIs

| Metric | Formula | Target |
|--------|---------|--------|
| Days to Appeal Submission | Avg(Appeal Date - Denial Date) | <10 days |
| Denial Resolution Time | Avg(Resolution Date - Denial Date) | <30 days |
| Write-Off Rate | Written Off $ / Denial $ | <10% |
| Cost Per Denial | Total Mgmt Cost / Denials Worked | <$43 |

---

## Root Cause Analysis

### 5 Whys Example

```
Problem: High CO-197 denial rate

Why 1: Claims submitted without authorization
Why 2: Authorization not obtained before service
Why 3: Scheduling staff unaware auth was required
Why 4: Auth requirements not in scheduling system
Why 5: No integration between payer rules and scheduling

Root Cause: Lack of automated auth requirement checking at scheduling
Solution: Implement payer-specific auth rules in scheduling workflow
```

### Pareto Analysis Approach

- Rank denial codes by frequency AND dollars
- Focus on top 20% causing 80% of denials
- Track intervention impact over time

---

## Quick Reference

### Top CARC Codes with Actions

| Code | Description | Action |
|------|-------------|--------|
| CO-45 | Exceeds fee schedule | Usually write-off (contractual) |
| CO-50 | Medical necessity | Appeal with clinical docs |
| CO-16 | Missing info | Correct and resubmit |
| CO-18 | Duplicate | Verify, add modifier if distinct |
| CO-22 | COB required | Obtain primary payer info |
| CO-97 | Bundled | Review NCCI, appeal if distinct |
| CO-197 | No authorization | Retroactive auth or appeal |
| PR-1 | Deductible | Patient responsibility |
| PR-2 | Coinsurance | Patient responsibility |
| PR-3 | Copay | Patient responsibility |

### Group Codes

| Code | Meaning | Financial Impact |
|------|---------|------------------|
| CO | Contractual Obligation | Provider write-off |
| PR | Patient Responsibility | Bill to patient |
| OA | Other Adjustment | Varies |
| PI | Payer-Initiated Reduction | Challenge if incorrect |
| CR | Correction/Reversal | Adjustment to prior payment |

---

## References

- [X12 Claim Adjustment Reason Codes (CARCs)](https://x12.org/codes/claim-adjustment-reason-codes)
- [X12 Remittance Advice Remark Codes (RARCs)](https://x12.org/codes/remittance-advice-remark-codes)
- [CMS Medicare Appeals](https://www.cms.gov/cciio/resources/fact-sheets-and-faqs/indexappealingdenials)
- [HFMA Denial Management Resources](https://www.hfma.org/)

---

## Related Skills

- **x12-835-payment** - CAS segment parsing, group codes
- **x12-edi-core** - 835 transaction structure
- **rcm-workflow-platform** - Routing engines, worklist management
- **rcm-low-balance-ar** - Low-value denial handling

---

## Related Skills

- **rcm-low-balance-ar** - Low-value denial handling, cost-to-collect decisions
- **rcm-workflow-platform** - Routing engines, worklist prioritization, queue management
- **x12-835-payment** - CAS segment parsing, remittance advice, group codes
- **x12-edi-core** - 835 transaction structure, envelopes, delimiters

---

**Document Version:** 1.1
**Last Updated:** 2026-01-20
**Source:** Healthcare RCM Denial Management Knowledge Base
