# ali-tax-practice-domain - Billing and Delivery Reference

This reference contains detailed billing calculations, delivery package specifications, signature workflows, and reminder rules.

---

## Delivery Package Workflow

### Package Components

| Component | Sort Order | Description |
|-----------|------------|-------------|
| COVER_LETTER | 1 | Summary letter to client |
| TAX_RETURN | 2 | Completed 1040/business return |
| FORM_8879 | 3 | E-file authorization (signature required) |
| INVOICE | 4 | Preparation fees |
| ESTIMATED_VOUCHER | 5 | Form 1040-ES for quarterly payments |
| STATE_VOUCHER | 6 | State estimated tax vouchers |

### Signature Workflow

```
Package Generated → Sent to Client → Viewed → Signed → Ready for Filing
                         │                        │
                         ▼                        ▼
                   Reminder Sent            Payment Collected
                   (days 3, 5, 7)
```

### Signature Constants

| Parameter | Value |
|-----------|-------|
| Reminder intervals | [3, 5, 7] days |
| Max reminders | 3 |
| Signing order | 8879 first (order=1) |

---

## Invoice & Billing

### Fee Calculation

```python
def calculate_total_fee(engagement):
    base = engagement.base_fee
    complexity = base * engagement.complexity_multiplier
    rush = RUSH_FEES.get(engagement.priority, 0)
    discount = engagement.discount_amount

    subtotal = base + complexity + rush - discount
    tax = subtotal * TAX_RATE if applicable
    return subtotal + tax
```

### Base Fees by Return Type

| Return Type | Base Fee |
|-------------|----------|
| 1040 (simple) | $200-300 |
| 1040 (itemized) | $400-600 |
| 1040 (investments) | $600-1000 |
| 1120 | $1000-2000 |
| 1120-S | $800-1500 |
| 1065 | $800-1500 |

---

## Complexity Multipliers

| Level | Multiplier | Example |
|-------|------------|---------|
| simple | 1.0x | Standard 1040 |
| moderate | 1.25x | Multiple schedules |
| complex | 1.5x | Investments, rentals |
| very_complex | 2.0x | Multi-state, K-1s, planning |

---

## Aging Buckets

| Bucket | Days Overdue |
|--------|--------------|
| Current | 0 |
| 1-30 | 1-30 |
| 31-60 | 31-60 |
| 61-90 | 61-90 |
| 90+ | >90 |

---

## Reminder/Escalation Rules

| Parameter | Value |
|-----------|-------|
| Default due days | 30 |
| Reminder days | [7, 14, 30] after due |
| Max reminders | 3 |
| Escalation threshold | 45 days overdue |

### Reminder Strategy

1. **Day 7**: Friendly reminder email
2. **Day 14**: Second reminder with payment link
3. **Day 30**: Final reminder before escalation
4. **Day 45**: Account suspended, collections notification
