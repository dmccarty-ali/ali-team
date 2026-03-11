# ali-tax-practice-domain - Return Types and Forms Reference

This reference contains comprehensive tables of return types, document types, extension procedures, estimated tax schedules, and the annual tax calendar.

---

## Return Types (Detailed)

### Individual Returns

| Form | Description | Due Date | Extension |
|------|-------------|----------|-----------|
| 1040 | Individual Income Tax | April 15 | Oct 15 (Form 4868) |
| 1040-SR | Seniors (65+) | April 15 | Oct 15 |
| 1040-NR | Nonresident Alien | April 15 | Oct 15 |
| 1040-X | Amended Return | 3 years from original | N/A |

### Business Returns

| Form | Entity Type | Due Date | Extension |
|------|-------------|----------|-----------|
| 1120 | C Corporation | April 15 (calendar year) | Oct 15 (Form 7004) |
| 1120-S | S Corporation | March 15 | Sept 15 (Form 7004) |
| 1065 | Partnership | March 15 | Sept 15 (Form 7004) |
| 1041 | Estate/Trust | April 15 | Sept 30 (Form 7004) |
| 990 | Tax-Exempt Org | May 15 (5th month after year-end) | Nov 15 (Form 8868) |

### State Returns

- Due dates vary by state (usually match federal or shortly after)
- Some states have no income tax (FL, TX, WA, etc.)
- Track each state's specific requirements

---

## Document Types

### Income Documents

| Document | Source | Data Extracted |
|----------|--------|----------------|
| W-2 | Employer | Wages, withholding, employer EIN |
| 1099-INT | Bank | Interest income |
| 1099-DIV | Brokerage | Dividends, capital gains distributions |
| 1099-B | Brokerage | Stock sales, cost basis |
| 1099-MISC | Various | Miscellaneous income |
| 1099-NEC | Clients | Nonemployee compensation |
| 1099-R | Retirement | Retirement distributions |
| 1099-G | Government | Unemployment, state refunds |
| 1099-K | Payment processors | Payment card/third-party transactions |
| K-1 (1065) | Partnership | Partner's share of income |
| K-1 (1120-S) | S Corp | Shareholder's share of income |
| K-1 (1041) | Trust | Beneficiary's share of income |

### Deduction Documents

| Document | Purpose |
|----------|---------|
| 1098 | Mortgage interest |
| 1098-T | Tuition payments |
| 1098-E | Student loan interest |
| Property tax statements | SALT deduction |
| Charitable receipts | Donation deductions |
| Medical receipts | Medical deductions (if >7.5% AGI) |

---

## Extension Filing

### Form 4868 (Individual)

- **Extends**: Filing deadline to October 15
- **Does NOT extend**: Payment deadline (April 15)
- **Estimate required**: Must estimate tax liability
- **Penalty**: If underpaid, interest accrues from April 15

### Form 7004 (Business)

- **For**: Corporations, partnerships, trusts
- **Extension**: 6 months (5 months for some entities)
- **Automatic**: No reason required

### Extension Tracking

```python
# Track extension status per client
class ExtensionStatus:
    NONE = "none"              # No extension filed
    FILED = "filed"            # Extension transmitted
    ACCEPTED = "accepted"      # IRS accepted extension
    REJECTED = "rejected"      # Fix and refile
    EXPIRED = "expired"        # Past extended deadline
```

---

## Estimated Tax Payments

### 1040-ES Schedule

| Quarter | Period Covered | Due Date |
|---------|----------------|----------|
| Q1 | Jan 1 - Mar 31 | April 15 |
| Q2 | Apr 1 - May 31 | June 15 |
| Q3 | Jun 1 - Aug 31 | September 15 |
| Q4 | Sep 1 - Dec 31 | January 15 (next year) |

### Safe Harbor Rules

To avoid underpayment penalty, pay the lesser of:
- 90% of current year tax, OR
- 100% of prior year tax (110% if AGI > $150k)

---

## Key Dates Calendar

### Annual Tax Calendar

| Date | Event |
|------|-------|
| January 15 | Q4 estimated tax due |
| January 31 | W-2s and 1099s due to recipients |
| March 15 | S-Corp (1120-S) and Partnership (1065) due |
| April 15 | Individual (1040) and C-Corp (1120) due; Q1 estimated due |
| April 18* | Actual deadline (if 15th is weekend/holiday) |
| June 15 | Q2 estimated tax due |
| September 15 | Extended S-Corp/Partnership due; Q3 estimated due |
| October 15 | Extended Individual/C-Corp due |

*Dates shift if falls on weekend or holiday
