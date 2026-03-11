---
name: ali-insurance-domain
description: |
  Property & casualty and medical malpractice insurance domain knowledge for analytics platforms. Use when:

  PLANNING: Designing insurance data models, architecting loss reserving systems, planning reinsurance reporting, considering regulatory compliance (NAIC)

  IMPLEMENTATION: Building policy administration interfaces, implementing claims workflows, creating reserve calculations, developing financial reporting

  GUIDANCE: Asking about insurance terminology (IBNR, case reserves, earned premium), policy structures, claims-made vs occurrence, reinsurance mechanics, NAIC schedules

  REVIEW: Validating insurance data calculations, checking reserve methodology, auditing premium earning patterns, verifying regulatory report completeness
---

# Insurance Domain

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing insurance data models (policies, claims, reserves)
- Architecting loss reserving systems
- Planning reinsurance reporting workflows
- Considering NAIC regulatory compliance
- Designing data structures for loss triangles

**Implementation:**
- Building policy administration interfaces
- Implementing claims workflows (FNOL through settlement)
- Creating reserve calculation systems
- Developing financial reporting (combined ratio, loss ratio)
- Building exposure and premium tracking

**Guidance/Best Practices:**
- Asking about insurance terminology (IBNR, case reserves, written/earned premium)
- Policy structures (declarations, conditions, exclusions, endorsements)
- Claims-made vs occurrence coverage
- Reinsurance mechanics (treaty, facultative, excess of loss)
- NAIC Annual Statement schedules
- Medical malpractice-specific concepts (tail coverage, consent-to-settle)

**Review/Validation:**
- Validating insurance data calculations
- Checking reserve methodology against actuarial standards
- Auditing premium earning patterns
- Verifying regulatory report completeness
- Detecting data quality issues (year-type mixing, system migrations)

---

## Key Principles

- **Accident year ≠ Policy year ≠ Calendar year**: Always explicitly specify year basis
- **Paid + Case + IBNR = Incurred**: The fundamental loss equation
- **Written premium earns over time**: Premium is not recognized immediately
- **Long-tail claims develop for years**: Medical malpractice claims can take 5-10 years to settle
- **Reinsurance net down**: Always present gross, ceded, and net amounts
- **Statutory (SAP) ≠ GAAP**: Two different accounting frameworks with different rules
- **Loss triangles require consistent basis**: Accident year, report year, or policy year — never mix
- **Defense costs matter**: Inside vs outside limits drastically affects loss calculation

---

## Property & Casualty Insurance Fundamentals

### Lines of Business

| Line | Description | Typical Coverage |
|------|-------------|------------------|
| **Commercial Lines** | Business insurance | General liability, commercial auto, workers' comp |
| **Personal Lines** | Individual insurance | Auto, homeowners, umbrella |
| **Specialty Lines** | Niche risks | Professional liability, cyber, D&O, E&O |
| **Professional Liability** | Errors & omissions | Medical malpractice, legal malpractice, accountants |

### Policy Structure

Every insurance policy contains:

1. **Declarations Page (Dec Page)**
   - Named insured
   - Policy number
   - Effective and expiration dates
   - Coverage limits
   - Premium amount
   - Deductibles/retentions

2. **Insuring Agreement**
   - What is covered (covered perils)
   - Coverage territory
   - Who is insured (named insured, additional insureds)

3. **Conditions**
   - Duties after loss (reporting requirements)
   - Cancellation provisions
   - Assignment and transfer
   - Legal action against insurer

4. **Exclusions**
   - What is NOT covered
   - Intentional acts
   - War, nuclear, terrorism (sometimes)
   - Contractual liability (sometimes)

5. **Endorsements**
   - Modifications to base policy
   - Additional covereds
   - Coverage extensions
   - Schedule of locations/vehicles/employees

### Policy Period Concepts

```
Policy Period: 2024-01-01 to 2024-12-31

Effective Date: 2024-01-01 00:01 AM (inception)
Expiration Date: 2024-12-31 11:59 PM (termination)

- Renewal: New policy begins immediately after expiration
- Mid-term endorsement: Changes coverage during policy period
- Cancellation: Early termination (pro-rata or short-rate refund)
```

### Premium Types

| Type | Definition | When Recognized |
|------|------------|-----------------|
| **Written Premium** | Total premium for policy issued | At policy inception |
| **Earned Premium** | Portion of written premium earned over time | Pro-rata by day |
| **Unearned Premium** | Written minus earned (liability on balance sheet) | Reserve until earned |
| **In-Force Premium** | Premium for policies currently active | As of specific date |
| **Direct Premium** | Premium before reinsurance | At inception |
| **Ceded Premium** | Premium paid to reinsurer | When reinsurance applies |
| **Net Premium** | Direct minus ceded | After reinsurance |

**Premium Earning Example:**

```python
# Annual policy: $12,000 written premium
# Effective: 2024-01-01
# As of: 2024-04-01 (90 days elapsed, 275 days remaining)

written_premium = 12000
days_in_policy = 365
days_elapsed = 90

earned_premium = written_premium * (days_elapsed / days_in_policy)
# earned_premium = 12000 * (90 / 365) = $2,959

unearned_premium = written_premium - earned_premium
# unearned_premium = 12000 - 2959 = $9,041
```

### Loss Types

| Type | Definition | Who Sets |
|------|------------|----------|
| **Paid Loss** | Money actually paid to claimant | Claims department |
| **Case Reserve** | Estimated remaining cost for known claim | Claims adjuster |
| **Incurred Loss** | Paid + Case Reserve | Calculated |
| **IBNR** | Incurred But Not Reported (unknown claims) | Actuary |
| **Ultimate Loss** | Final expected cost (Incurred + IBNR) | Actuary |

**Loss Equation:**

```
Ultimate Loss = Paid Loss + Case Reserve + IBNR

Or alternatively:
Ultimate Loss = Reported Incurred + IBNR
where Reported Incurred = Paid + Case
```

---

## Medical Malpractice Insurance (MLMIC Focus)

Medical malpractice insurance is long-tail, high-severity professional liability coverage for healthcare providers.

### Claims-Made vs Occurrence Policies

| Aspect | Claims-Made | Occurrence |
|--------|-------------|------------|
| **Trigger** | Claim reported during policy period | Incident occurs during policy period |
| **Coverage duration** | Only while policy active | Perpetual (even after cancellation) |
| **Cost** | Lower initially, increases with maturity | Higher upfront |
| **Tail exposure** | Requires tail coverage if cancelled | No tail needed |
| **Retroactive date** | Coverage only for incidents after retro date | No retro date concept |

**Claims-Made Example:**

```
Policy Period: 2024-01-01 to 2024-12-31
Retroactive Date: 2020-01-01

Covered incident: Surgery on 2022-05-15, claim reported 2024-03-10
- Incident after retroactive date: ✓
- Claim reported during policy period: ✓
- Result: COVERED

Not covered incident: Surgery on 2023-08-20, claim reported 2025-02-15
- Incident after retroactive date: ✓
- Claim reported during policy period: ✗ (after expiration)
- Result: NOT COVERED (unless tail coverage purchased)
```

### Tail Coverage (Extended Reporting Period)

**Purpose:** Extends reporting period after claims-made policy cancels/non-renews

**Types:**

| Type | Duration | Cost | When Used |
|------|----------|------|-----------|
| **Basic Tail** | 1-3 years | 1-2x annual premium | Temporary coverage gap |
| **Extended Tail** | 5-10 years | 2-3x annual premium | Retirement, career change |
| **Unlimited Tail** | Forever | 3-5x annual premium | Permanent departure from medicine |

**Tail Cost Example:**

```python
# Physician retiring after 20 years with same carrier
annual_premium = 15000  # Last year's premium
tail_multiplier = 3.0   # Unlimited tail

tail_cost = annual_premium * tail_multiplier
# tail_cost = $45,000 one-time payment
```

### Prior Acts Coverage (Nose Coverage)

**Purpose:** Covers incidents that occurred before policy inception (when switching carriers)

**Requirements:**
- No gap in coverage
- Proof of prior coverage
- No known claims or incidents

**Example:**

```
Previous carrier: Policy expired 2024-12-31
New carrier: Policy effective 2025-01-01
Retroactive date: 2015-01-01 (continuous coverage)

Surgery performed: 2018-06-10 (under old carrier's policy)
Claim reported: 2025-03-15 (under new carrier's policy)

With prior acts coverage: COVERED by new carrier
Without prior acts coverage: NOT COVERED (no active policy when claim reported)
```

### Consent-to-Settle Clauses

**Purpose:** Gives insured physician control over settlement decisions

**Types:**

| Clause Type | Carrier Can | Insured Can | Common In |
|-------------|-------------|-------------|-----------|
| **Hammer Clause** | Settle against insured's wishes | Refuse, but pays excess above recommended | Commercial policies |
| **Absolute Consent** | Never settle without consent | Refuse settlement, carrier pays all costs | Physician-owned carriers |
| **Modified Consent** | Settle if insured unreasonably refuses | Refuse, subject to reasonableness review | Balanced policies |

**Consent-to-Settle Data Model:**

```python
class Claim:
    settlement_offered: bool
    settlement_amount: Decimal
    settlement_recommended_by_carrier: bool
    insured_consent_required: bool
    insured_consent_given: Optional[bool]
    consent_requested_date: Optional[date]
    consent_response_date: Optional[date]
    hammer_clause_triggered: bool
    insured_excess_liability: Decimal  # If hammer clause invoked
```

### Defense Costs (Inside vs Outside Limits)

**Inside Limits:**
```
Policy Limit: $1,000,000 per claim
Defense Costs: $200,000
Settlement/Judgment: $900,000
Total Paid: $1,100,000

Result: Insured pays $100,000 excess (defense ate into limit)
```

**Outside Limits:**
```
Policy Limit: $1,000,000 per claim
Defense Costs: $200,000 (paid in addition to limit)
Settlement/Judgment: $900,000
Total Paid: $1,100,000

Result: Insured pays $0 (defense paid on top of limit)
```

### Medical Malpractice Loss Drivers

| Factor | Impact on Losses |
|--------|------------------|
| **Specialty** | High-risk (neurosurgery, OB/GYN) vs low-risk (dermatology, pediatrics) |
| **Geography** | High-award jurisdictions (NY, CA) vs tort reform states (TX, FL) |
| **Provider experience** | Higher early-career losses, stabilizes after 5-10 years |
| **Hospital affiliation** | Teaching hospitals = higher exposure |
| **Procedure volume** | More procedures = more exposure |
| **Prior claims** | Claims history strong predictor of future claims |

### Long-Tail Development

Medical malpractice claims develop slowly:

| Time Since Incident | % of Claims Reported | % of Claims Settled |
|---------------------|----------------------|---------------------|
| 1 year | 30% | 5% |
| 2 years | 55% | 15% |
| 3 years | 70% | 30% |
| 5 years | 85% | 55% |
| 7 years | 92% | 75% |
| 10 years | 97% | 90% |

**Implication:** Reserves must be held for 5-10 years. IBNR calculations require long development patterns.

---

## Claims Lifecycle

### First Notice of Loss (FNOL)

**Intake Information:**
- Date of incident (loss date)
- Date claim reported
- Claimant information
- Brief description of incident
- Severity indicators (injury type, hospitalization)
- Policy information (policy number, limits)

**Medical Malpractice FNOL Example:**

```python
class FNOL:
    claim_number: str
    report_date: date
    incident_date: date  # Surgery/treatment date
    reporting_lag_days: int  # report_date - incident_date

    claimant_name: str
    claimant_injury: str  # "Birth injury", "Surgical complication"
    claimant_age: int

    insured_provider: str
    insured_specialty: str
    procedure_type: str

    policy_number: str
    policy_limits: str  # "1M/3M" = $1M per claim, $3M aggregate
    coverage_type: str  # "claims_made" or "occurrence"
    retroactive_date: Optional[date]
```

### Claim Investigation and Adjustment

**Claims Adjuster Responsibilities:**
- Investigate facts of loss
- Determine coverage (is claim covered under policy?)
- Establish initial case reserve
- Assign defense counsel (for med mal)
- Coordinate medical reviews/expert opinions

**Coverage Determination Checklist:**
- [ ] Incident date within policy period (occurrence) or retroactive date (claims-made)?
- [ ] Claim reported within reporting period (claims-made)?
- [ ] Named insured or additional insured?
- [ ] Excluded activity? (intentional acts, criminal acts)
- [ ] Policy in force at time of incident/report?
- [ ] Limits available (not exhausted by prior claims)?

### Case Reserving

**Reserve Philosophy:**

Case reserves represent the adjuster's best estimate of remaining claim costs (indemnity + defense).

**Reserve Adequacy Levels:**

| Adequacy | Meaning | Typical Target |
|----------|---------|----------------|
| **Optimistic** | Likely to increase | Avoid |
| **Adequate** | Best estimate, 50/50 probability | Industry standard |
| **Conservative** | Likely to decrease | High-stakes or uncertain claims |
| **Redundant** | Overly conservative | Regulatory scrutiny |

**Reserve Change Workflow:**

```python
class ReserveChange:
    claim_id: str
    previous_reserve: Decimal
    new_reserve: Decimal
    change_amount: Decimal
    change_reason: str  # "Discovery", "Settlement negotiation", "New medical info"
    changed_by: str     # Adjuster name
    changed_at: datetime
    supervisor_approved: bool
```

### Subrogation and Salvage

**Subrogation:** Recovering paid losses from responsible third parties

**Example:** Surgical error partially caused by defective medical device → Insurer pays claim, then sues device manufacturer

**Salvage:** Recovering value from damaged property

**Salvage in Med Mal:** Rare (no property damage), but may recover costs from co-defendants

### Claim Settlement and Closure

**Settlement Types:**

| Type | Description | Releases |
|------|-------------|----------|
| **Full and Final** | Total settlement, case closed | All claims, all parties |
| **Partial Settlement** | Settles some claims/defendants | Specific claims only |
| **Structured Settlement** | Payments over time (annuity) | Upon completion |
| **High-Low Agreement** | Range agreement, jury decides amount | Conditional |

**Closure Checklist:**
- [ ] Settlement agreement signed
- [ ] Release obtained from claimant
- [ ] Payment issued and cleared
- [ ] Reserve reduced to zero
- [ ] File closed in system
- [ ] Subrogation potential evaluated

### Reopened Claims

**Reasons for Reopening:**
- Additional damages discovered
- Settlement payment issues
- Supplemental medical treatment
- Bad faith allegation

**Reopening Data Model:**

```python
class Claim:
    status: str  # "open", "closed", "reopened"
    closed_date: Optional[date]
    reopen_date: Optional[date]
    reopen_reason: Optional[str]
    reopen_reserve: Decimal  # New reserve on reopen
```

---

## Underwriting

### Risk Selection and Classification

**Underwriting Process:**

```
Application → Initial Review → Risk Assessment → Pricing → Quote → Bind
     │              │                │              │        │       │
     │              │                │              │        │       │
     ▼              ▼                ▼              ▼        ▼       ▼
Provider info  Coverage check  Loss history   Rate calc  Accept  Issue policy
Medical license Limits requested   Specialty  Experience mod  Decline
Claims history  Retro date        Territory    Schedule rating  Terms
```

**Medical Malpractice Underwriting Factors:**

| Factor | Impact on Rate |
|--------|----------------|
| **Specialty** | Base rate (neurosurgeon vs pediatrician) |
| **Territory** | State/county litigation environment |
| **Limits** | Higher limits = higher premium (but less than linear) |
| **Experience** | Years in practice, claims history |
| **Coverage type** | Claims-made maturity or occurrence |
| **Hospital privileges** | Teaching vs community hospital |
| **Part-time vs full-time** | Procedure volume adjustment |

### Rating Factors and Premium Calculation

**Base Rate:** Premium per $1M coverage for specialty in territory

**Example Base Rates (per $1M / $3M limits):**

| Specialty | Territory | Base Rate |
|-----------|-----------|-----------|
| Internal Medicine | New York | $15,000 |
| OB/GYN | New York | $90,000 |
| Neurosurgery | New York | $150,000 |
| Dermatology | New York | $8,000 |

**Premium Calculation:**

```python
def calculate_premium(specialty: str, territory: str, limits: tuple,
                      claims_made_maturity: int, experience_mod: float,
                      schedule_rating_credit: float):
    """
    Calculate medical malpractice premium.

    limits: (per_claim_limit, aggregate_limit) in millions
    claims_made_maturity: years on claims-made (0-5+)
    experience_mod: 0.75 to 1.50 (claims history adjustment)
    schedule_rating_credit: -0.25 to +0.25 (underwriter discretion)
    """
    base_rate = BASE_RATES[specialty][territory]

    # Limits adjustment (if not standard $1M/$3M)
    limits_factor = calculate_limits_factor(limits)

    # Claims-made maturity discount (lower in early years)
    maturity_factor = MATURITY_DISCOUNTS[min(claims_made_maturity, 5)]

    # Experience modification
    # 0.75 = 25% credit for excellent history
    # 1.50 = 50% surcharge for poor history

    # Schedule rating (underwriter judgment)
    schedule_factor = 1.0 + schedule_rating_credit

    premium = (base_rate * limits_factor * maturity_factor *
               experience_mod * schedule_factor)

    return premium

# Example:
# OB/GYN, NY, $1M/$3M, 3rd year claims-made, good history
premium = calculate_premium(
    specialty="OB/GYN",
    territory="New York",
    limits=(1, 3),
    claims_made_maturity=3,
    experience_mod=0.85,  # 15% credit
    schedule_rating_credit=-0.05  # 5% credit
)
# premium ≈ $90,000 * 1.0 * 0.90 * 0.85 * 0.95 = $65,385
```

**Claims-Made Maturity Discounts:**

| Year | Maturity Factor | Why |
|------|-----------------|-----|
| 1 | 0.40 | Only 1 year of retro exposure |
| 2 | 0.60 | 2 years of retro exposure |
| 3 | 0.75 | 3 years of retro exposure |
| 4 | 0.85 | 4 years of retro exposure |
| 5+ | 1.00 | Full maturity ("mature rate") |

### Experience Modification

**Experience Mod Formula (simplified):**

```
Experience Mod = (Actual Losses / Expected Losses) * Credibility + 1.0 * (1 - Credibility)

Where:
- Actual Losses: Physician's historical losses
- Expected Losses: Average losses for specialty/territory
- Credibility: 0.0 to 1.0 (higher for more years of data)
```

**Example:**

```python
actual_losses = 500000  # Physician had $500K in claims over 5 years
expected_losses = 300000  # Expected $300K for specialty
credibility = 0.75  # 5 years of data = 75% credible

experience_mod = (actual_losses / expected_losses) * credibility + 1.0 * (1 - credibility)
# = (500000 / 300000) * 0.75 + 1.0 * 0.25
# = 1.667 * 0.75 + 0.25
# = 1.25 + 0.25 = 1.50

# Result: 50% surcharge due to poor claims history
```

### Schedule Rating

**Purpose:** Underwriter discretionary adjustment for factors not captured in rating algorithm

**Schedule Rating Factors (Examples):**

| Factor | Typical Range |
|--------|---------------|
| Risk management program | -10% to 0% |
| Continuing education | -5% to 0% |
| Electronic health records | -5% to 0% |
| Poor communication skills | 0% to +10% |
| Inadequate office staff | 0% to +10% |
| Prior license discipline | 0% to +20% |

**Total Schedule Rating:** Typically limited to ±25%

---

## Reinsurance

### Treaty vs Facultative

| Aspect | Treaty Reinsurance | Facultative Reinsurance |
|--------|-------------------|------------------------|
| **Scope** | All policies meeting criteria | Individual policy/risk |
| **Negotiation** | Annual contract | Per-risk negotiation |
| **Binding** | Automatic for qualifying risks | Reinsurer can decline |
| **Cost** | Lower (portfolio pricing) | Higher (individual pricing) |
| **Use case** | Routine capacity needs | Large/unusual risks |

### Proportional Reinsurance

**Quota Share:**

Reinsurer takes fixed percentage of every policy.

```
Quota Share: 30%

Original Policy: $1M limit, $10K premium, $50K loss

Ceded to Reinsurer:
- Premium: $10K * 30% = $3K
- Loss: $50K * 30% = $15K
- Net Retained: $10K - $3K = $7K premium, $50K - $15K = $35K loss
```

**Surplus Share:**

Reinsurer takes variable percentage based on policy limits.

```
Insurer Retention: $500K
Surplus Lines: 5 (insurer can cede up to 5x retention = $2.5M)

Policy: $2M limit, $20K premium

Retention: $500K / $2M = 25%
Ceded: $1.5M / $2M = 75%

Ceded Premium: $20K * 75% = $15K
If Loss: Loss shared 25% / 75%
```

### Non-Proportional Reinsurance

**Excess of Loss:**

Reinsurer pays losses exceeding attachment point.

```
Per-Claim Excess: $250K xs $250K (excess of $250K, limit $250K)

Loss Scenario 1: $100K claim
- Insurer pays: $100K (below attachment)
- Reinsurer pays: $0

Loss Scenario 2: $400K claim
- Insurer pays: $250K (retention)
- Reinsurer pays: $150K (above retention, within limit)

Loss Scenario 3: $600K claim
- Insurer pays: $250K (retention) + $100K (excess of reinsurer limit)
- Reinsurer pays: $250K (full reinsurance limit)
```

**Aggregate Excess:**

Reinsurer pays when total losses exceed threshold.

```
Aggregate Excess: $2M xs $5M (attachment $5M, limit $2M)

Annual Losses:
- Q1: $1M
- Q2: $1.5M
- Q3: $2M
- Q4: $1.2M
Total: $5.7M

Insurer pays: $5M (attachment point)
Reinsurer pays: $700K (losses above $5M)
```

**Catastrophe Reinsurance:**

Protects against accumulation of losses from single event.

```
Cat XL: $10M xs $5M

Hurricane damages multiple insured properties:
- 50 homeowner claims totaling $8M

Insurer pays: $5M
Reinsurer pays: $3M
```

### Reinsurance Program Structure

**Layered Program Example (Medical Malpractice Insurer):**

```
Layer 1: $1M xs $1M (first excess layer)
Layer 2: $2M xs $2M (second excess layer)
Layer 3: $5M xs $4M (third excess layer)
Layer 4: $10M xs $9M (fourth excess layer)

Total Coverage: $18M xs $1M (covers losses from $1M to $19M)

Example Claim: $7M settlement

Breakdown:
- Insurer retention: $1M
- Layer 1 pays: $1M (exhausted)
- Layer 2 pays: $2M (exhausted)
- Layer 3 pays: $3M (partial, $2M remaining)
- Layer 4 pays: $0 (not reached)
```

### Ceded vs Assumed vs Net

```
Direct Premium Written: $50M (premium insurer writes)
Ceded Premium: -$15M (premium paid to reinsurers)
Assumed Premium: +$2M (premium from other insurers as reinsurer)
Net Premium Written: $37M (direct - ceded + assumed)

Direct Losses: $30M
Ceded Losses: -$9M (reinsurer pays this)
Assumed Losses: +$1M (losses on assumed premium)
Net Losses: $22M
```

### Reinsurance Recoverable

**Balance Sheet Liability:**

Reinsurance recoverable = amounts owed by reinsurers for paid and unpaid claims.

```python
class ReinsuranceRecoverable:
    paid_loss_recoverable: Decimal  # Reinsurer owes for already paid claims
    case_reserve_recoverable: Decimal  # Reinsurer's share of case reserves
    ibnr_recoverable: Decimal  # Reinsurer's share of IBNR

    total_recoverable: Decimal  # Sum of above

    allowance_for_uncollectible: Decimal  # Provision for reinsurer default

    net_recoverable: Decimal  # total - allowance
```

**Reinsurer Credit Risk:**

Insurers must evaluate reinsurer financial strength (AM Best rating) and set allowances for potential non-payment.

---

## Financial Reporting and Regulatory

### NAIC Annual Statement

**Statutory Accounting Principles (SAP)** require annual filing with state insurance departments.

**Key Schedules:**

| Schedule | Content | Purpose |
|----------|---------|---------|
| **Schedule P** | Loss and LAE reserves by line | Reserve adequacy analysis |
| **Schedule F** | Reinsurance | Ceded and assumed business detail |
| **Schedule T** | Premiums and losses by territory | Geographic analysis |
| **Schedule Y** | Investments | Asset portfolio detail |

### Schedule P Structure

**Loss Development Triangle Format:**

Schedule P Part 2 shows historical loss development by accident year.

```
Accident Year | 12 mos | 24 mos | 36 mos | 48 mos | 60 mos | ... | Current
--------------------------------------------------------------------------
2020          | 5,000  | 8,000  | 9,500  | 10,200 | 10,500 |     | 10,800
2021          | 4,500  | 7,200  | 9,000  | 9,800  | 10,200 |     | 10,200
2022          | 5,200  | 8,500  | 10,300 | 11,000 |        |     | 11,000
2023          | 4,800  | 7,800  | 9,200  |        |        |     | 9,200
2024          | 5,500  | 8,200  |        |        |        |     | 8,200

(Amounts in thousands, losses incurred)
```

**Schedule P Part 3: Claims Counts**

Shows number of claims reported and closed by accident year.

**Note:** See ali-actuarial skill for loss development methods (chain ladder, Bornhuetter-Ferguson).

### Statutory (SAP) vs GAAP Accounting

| Aspect | Statutory (SAP) | GAAP |
|--------|-----------------|------|
| **Purpose** | Solvency (can insurer pay claims?) | Profitability (is company making money?) |
| **Regulator** | State insurance departments | SEC, FASB |
| **Philosophy** | Conservative (protect policyholders) | Matching principle (revenue with expenses) |
| **Acquisition costs** | Expensed immediately | Deferred and amortized |
| **Non-admitted assets** | Excluded from surplus | Included in assets |
| **Discounting** | Not allowed (most cases) | Allowed/required for long-tail |

**Example Difference:**

```
New policy written: $10K premium, $2K commission

SAP Treatment:
- Revenue: $10K premium (pro-rata earned)
- Expense: $2K commission (immediate)
- First-year profitability: Depressed

GAAP Treatment:
- Revenue: $10K premium (pro-rata earned)
- Expense: $2K commission (deferred, amortized with premium earning)
- First-year profitability: Matched
```

### Combined Ratio, Loss Ratio, Expense Ratio

**Loss Ratio:**

```
Loss Ratio = Incurred Losses / Earned Premium

Example:
Earned Premium: $10M
Incurred Losses: $7M
Loss Ratio = $7M / $10M = 70%
```

**Loss Adjustment Expense (LAE) Ratio:**

```
LAE Ratio = Loss Adjustment Expenses / Earned Premium

LAE types:
- Allocated LAE (ALAE): Directly attributable to claims (defense counsel, experts)
- Unallocated LAE (ULAE): Claims department overhead

Example:
ALAE: $1M
ULAE: $500K
Earned Premium: $10M
LAE Ratio = $1.5M / $10M = 15%
```

**Expense Ratio:**

```
Expense Ratio = Underwriting Expenses / Written Premium

Underwriting expenses: commissions, premium taxes, overhead

Example:
Commissions: $1M
Premium taxes: $200K
Overhead: $800K
Written Premium: $10M
Expense Ratio = $2M / $10M = 20%
```

**Combined Ratio:**

```
Combined Ratio = Loss Ratio + LAE Ratio + Expense Ratio

Example:
Loss Ratio: 70%
LAE Ratio: 15%
Expense Ratio: 20%
Combined Ratio = 70% + 15% + 20% = 105%

Interpretation:
- <100%: Underwriting profit
- =100%: Break-even
- >100%: Underwriting loss (must make up with investment income)
```

### Investment Income and Surplus

**Insurers earn money two ways:**

1. **Underwriting profit** (combined ratio <100%)
2. **Investment income** (on premiums held before paying claims)

**Policyholder Surplus:**

```
Surplus = Assets - Liabilities

Assets: Cash, investments, reinsurance recoverable, premium receivable
Liabilities: Loss reserves (case + IBNR), unearned premium, ceded premium payable

Example:
Assets: $100M
Loss reserves: $60M
Unearned premium: $20M
Liabilities: $80M
Surplus: $100M - $80M = $20M

Surplus represents cushion to absorb adverse loss development.
```

### Risk-Based Capital (RBC)

**Purpose:** Minimum capital requirement based on insurer's risk profile

**RBC Formula Components:**

- R0: Asset risk - affiliates
- R1: Fixed income asset risk
- R2: Equity asset risk
- R3: Credit risk (reinsurance recoverable)
- R4: Reserve risk (loss reserve adequacy)
- R5: Written premium risk (underwriting risk)

**RBC Ratio:**

```
RBC Ratio = Total Adjusted Capital / Authorized Control Level RBC

Interpretation:
- >200%: No regulatory action
- 150-200%: Company Action Level (must submit plan)
- 100-150%: Regulatory Action Level (regulator may intervene)
- 70-100%: Authorized Control Level (regulator can take control)
- <70%: Mandatory Control Level (regulator must take control)
```

### State Insurance Department Regulation

**Key State Powers:**

- Approve/disapprove rates (prior approval states vs file-and-use)
- License insurers (certificate of authority)
- Financial examination (every 3-5 years)
- Market conduct examination (compliance with laws)
- Receivership (insolvent insurers)

**Residual Markets:**

When private market won't cover high-risk insureds, state-run pools provide coverage:
- Workers' Compensation: Assigned Risk Pool
- Medical Malpractice: Joint Underwriting Association (JUA) in some states
- Homeowners (coastal): FAIR Plans

---

## Insurance Data Concepts

### Policy Administration System Data

**Core Entities:**

```python
class Policy:
    policy_number: str
    named_insured: str
    effective_date: date
    expiration_date: date

    coverage_type: str  # "claims_made", "occurrence"
    retroactive_date: Optional[date]

    per_claim_limit: Decimal
    aggregate_limit: Decimal
    retention: Decimal  # Deductible or self-insured retention

    written_premium: Decimal
    premium_payment_plan: str  # "annual", "quarterly", "monthly"

    territory: str
    line_of_business: str
    specialty: Optional[str]  # For professional liability
```

**Endorsements:**

```python
class Endorsement:
    policy_id: str
    endorsement_number: str
    effective_date: date
    transaction_type: str  # "new_business", "endorsement", "cancellation", "renewal"

    premium_change: Decimal  # Can be negative (return premium)
    limit_change: Optional[Decimal]
    coverage_change_description: str
```

### Claims System Data

**Core Entities:**

```python
class Claim:
    claim_number: str
    policy_id: str

    incident_date: date
    report_date: date
    reporting_lag_days: int

    claimant_name: str
    claimant_type: str  # "patient", "property_owner", "injured_worker"

    loss_type: str  # "indemnity", "medical_only", "expense_only"
    injury_severity: str  # "minor", "major", "catastrophic"

    paid_loss: Decimal
    paid_alae: Decimal
    paid_ulae: Decimal

    case_reserve_indemnity: Decimal
    case_reserve_alae: Decimal

    status: str  # "open", "closed", "reopened"
    close_date: Optional[date]

    reinsurance_recoverable: Decimal
```

### Exposure Measures

**Purpose:** Denominator for loss ratio calculations

| Exposure Measure | Use Case |
|------------------|----------|
| **Earned Premium** | Overall loss ratio (incurred loss / earned premium) |
| **Policy Count** | Frequency (claim count / policy count) |
| **Earned Exposures** | Adjusted for policy size (house-years, car-years) |
| **Payroll** | Workers' compensation (losses per $100 of payroll) |
| **Sales** | General liability (losses per $1M of sales) |

**Example:**

```python
# Medical malpractice: exposure = physician-years
policies = [
    {"physician_count": 1, "months_active": 12},  # 1.0 physician-year
    {"physician_count": 2, "months_active": 6},   # 1.0 physician-year
    {"physician_count": 1, "months_active": 3},   # 0.25 physician-year
]

total_exposure = sum(p["physician_count"] * p["months_active"] / 12 for p in policies)
# total_exposure = 1.0 + 1.0 + 0.25 = 2.25 physician-years

# Frequency calculation
claim_count = 3
frequency = claim_count / total_exposure
# frequency = 3 / 2.25 = 1.33 claims per physician-year
```

### Accident Year, Policy Year, Calendar Year, Report Year

**Critical distinction:** Choose ONE basis and stick with it.

**Accident Year (AY):**
```
Losses grouped by date of incident

Example: AY 2024 = all incidents occurring Jan 1 - Dec 31, 2024
- May be reported in 2024, 2025, 2026, etc.
- May be paid in 2024, 2025, 2026, etc.
```

**Policy Year (PY):**
```
Losses grouped by policy effective date

Example: PY 2024 = all losses under policies effective in 2024
- Incident may occur anytime during policy period
- For 12-month policies, spans 2024-2025
```

**Calendar Year (CY):**
```
Losses grouped by payment date or accounting period

Example: CY 2024 = all losses paid or booked in 2024
- Incident may have occurred years earlier
- Used for financial statement reporting
```

**Report Year (RY):**
```
Losses grouped by date claim was reported

Example: RY 2024 = all claims reported in 2024
- Common for claims-made policies
- Incident may have occurred before 2024 (if within retro date)
```

**Why This Matters:**

```
Mixing year bases INVALIDATES loss triangles and trend analysis.

Example Error:
- Triangle rows are accident year
- Triangle columns are calendar year development
- Accident year 2020 at 24 months = losses as of Dec 31, 2021
- But some 2020 policies extend into 2021, creating mismatched cohorts
```

### Statistical Plan Data

**Purpose:** Aggregated data submitted to rating bureaus (NCCI, ISO) for industrywide rate analysis

**Unit Statistical Records:**

```python
class StatisticalRecord:
    policy_id: str
    exposure_period_start: date
    exposure_period_end: date

    territory_code: str
    class_code: str  # ISO/NCCI classification
    exposure_basis: str  # "premium", "payroll", "sales"
    exposure_amount: Decimal

    premium: Decimal
    losses: Decimal  # Typically accident date basis, valued at report date

    limit_indicator: str
    deductible_indicator: str
```

### Loss Triangles as Data Structures

**Purpose:** Track loss development over time for reserving

**Triangle Dimensions:**

```
Rows: Accident Year (or Policy Year, Report Year)
Columns: Development Period (12 months, 24 months, ...)
Cells: Cumulative Incurred Losses (or Paid Losses)
```

**Data Structure Options:**

**Option 1: Long Format (Database-Friendly)**

```python
class LossDevelopment:
    accident_year: int
    valuation_date: date
    development_months: int
    incurred_loss: Decimal
    paid_loss: Decimal
    claim_count: int
    case_reserve: Decimal
```

**Option 2: Wide Format (Triangle Display)**

```python
# Dictionary structure
triangle = {
    2020: {12: 5000, 24: 8000, 36: 9500, 48: 10200, 60: 10500},
    2021: {12: 4500, 24: 7200, 36: 9000, 48: 9800},
    2022: {12: 5200, 24: 8500, 36: 10300},
    2023: {12: 4800, 24: 7800},
    2024: {12: 5500},
}
```

**Note:** See ali-actuarial skill for triangle analysis methods (age-to-age factors, development patterns).

### Data Quality Issues Common in Insurance

| Issue | Why It Happens | Impact | Detection |
|-------|----------------|--------|-----------|
| **Year-type mixing** | Switching from AY to PY basis | Invalidates trends | Check for cohort inconsistency |
| **Coverage changes** | Policy form updates, limit changes | Non-comparable exposures | Version control on policy forms |
| **Mergers/acquisitions** | Company combines with another | Different reserving practices | Check for step changes in reserves |
| **System migrations** | Move from old to new claims system | Data mapping errors, lost history | Validate pre/post migration totals |
| **Reinsurance netting** | Sometimes gross, sometimes net | Inconsistent loss amounts | Always store gross, ceded, net |
| **Late reported claims** | IBNR becomes known | Historical data changes | Immutable snapshots at each valuation |
| **Large loss treatment** | Cap, exclude, or layer | Distorts severity trends | Separate large loss analysis |

---

## Key Insurance Metrics

### Loss Ratio (Pure, Gross, Net)

**Pure Loss Ratio (Losses Only):**

```
Pure Loss Ratio = Incurred Losses / Earned Premium
```

**Gross Loss Ratio (Losses + ALAE):**

```
Gross Loss Ratio = (Incurred Losses + ALAE) / Earned Premium
```

**Net Loss Ratio (After Reinsurance):**

```
Net Loss Ratio = (Net Incurred Losses + Net ALAE) / Net Earned Premium

Where:
Net Incurred = Gross Incurred - Ceded Recoverable
Net Earned Premium = Gross Earned - Ceded Earned
```

### Frequency and Severity

**Frequency (Claim Rate):**

```
Frequency = Claim Count / Exposure

Example:
Claim Count: 45
Physician-Years: 300
Frequency = 45 / 300 = 0.15 claims per physician-year
```

**Severity (Average Claim Size):**

```
Severity = Incurred Losses / Claim Count

Example:
Incurred Losses: $6,750,000
Claim Count: 45
Severity = $6,750,000 / 45 = $150,000 per claim
```

**Pure Premium (Loss Cost):**

```
Pure Premium = Frequency × Severity

Or directly:
Pure Premium = Incurred Losses / Exposure

Example:
Frequency: 0.15 per physician-year
Severity: $150,000
Pure Premium = 0.15 × $150,000 = $22,500 per physician-year

Check:
Incurred Losses: $6,750,000
Exposure: 300 physician-years
Pure Premium = $6,750,000 / 300 = $22,500 ✓
```

### Retention Ratio

**Purpose:** Measure how much premium insurer keeps vs cedes to reinsurers

```
Retention Ratio = Net Written Premium / Gross Written Premium

Example:
Gross Written: $50M
Ceded: $15M
Net Written: $35M
Retention Ratio = $35M / $50M = 70%

Interpretation: Insurer retains 70% of premium (cedes 30%)
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Mixing accident year and calendar year | Invalidates loss development | Pick one year basis and document it |
| Ignoring reporting lag | Early months look optimistic | Always show development age/lag |
| Comparing gross to net ratios | Apples to oranges | Always specify gross vs net |
| Forgetting defense costs in limits | Understates losses | Clarify inside vs outside limits |
| Treating IBNR as actual reserves | IBNR is an estimate | Separate case reserves from IBNR |
| Using written premium for loss ratio | Premium not earned yet | Use earned premium for loss ratio |
| Combining short-tail and long-tail | Different development patterns | Separate by tail length |
| Not adjusting for large losses | Large losses distort trends | Cap or exclude for trending |
| Assuming linear earning | Some policies earn differently | Use actual earning curves |
| Ignoring policy form changes | Coverage changes affect comparability | Track form versions and adjust |

---

## Integration with Other Skills

### ali-actuarial

For deep dive on:
- Loss development methods (chain ladder, Bornhuetter-Ferguson, Cape Cod)
- IBNR calculation techniques
- Reserve adequacy testing
- Triangle analysis and diagnostics
- Tail factors

### ali-data-architecture

For:
- Dimensional modeling of insurance data (star schema for claims/policies)
- Slowly changing dimensions (policy forms, territories)
- Data warehouse design for insurance analytics

### ali-snowflake-core

For:
- Implementing insurance analytics in Snowflake
- Time-series queries for loss development
- Windowing functions for lag calculations

---

## Quick Reference

### Premium Accounting Equation

```
Written Premium = Earned Premium + Change in Unearned Premium
```

### Loss Accounting Equation

```
Ultimate Loss = Paid Loss + Case Reserve + IBNR
```

### Combined Ratio

```
Combined Ratio = (Incurred Loss + LAE) / Earned Premium + Expense / Written Premium
```

### Experience Modification

```
Mod = [(Actual Loss / Expected Loss) × Credibility] + [1.0 × (1 - Credibility)]
```

### Reinsurance Recoverable

```
Recoverable = Paid Recoverable + Case Reserve Recoverable + IBNR Recoverable
```

### Frequency and Severity

```
Frequency = Claim Count / Exposure
Severity = Incurred Loss / Claim Count
Pure Premium = Frequency × Severity
```

---

## Common Abbreviations

| Abbrev | Meaning |
|--------|---------|
| **IBNR** | Incurred But Not Reported |
| **ALAE** | Allocated Loss Adjustment Expense |
| **ULAE** | Unallocated Loss Adjustment Expense |
| **LAE** | Loss Adjustment Expense |
| **DCC** | Defense and Cost Containment |
| **DWP** | Direct Written Premium |
| **NWP** | Net Written Premium |
| **DEP** | Direct Earned Premium |
| **NEP** | Net Earned Premium |
| **AY** | Accident Year |
| **PY** | Policy Year |
| **CY** | Calendar Year |
| **RY** | Report Year |
| **XOL** | Excess of Loss |
| **QS** | Quota Share |
| **WC** | Workers' Compensation |
| **GL** | General Liability |
| **MPL** | Medical Professional Liability (Medical Malpractice) |

---

## References

- [NAIC Annual Statement Instructions](https://content.naic.org/industry/annual-financial-reporting.htm) - Regulatory reporting guidance
- [ISO Circulars](https://www.verisk.com/insurance/products/iso-circulars/) - Standard policy forms and rates
- [CAS Exam Syllabus](https://www.casact.org/exams-admissions) - Actuarial foundations
- [Medical Malpractice Insurers Association](https://www.advisenltd.com/data/professional-liability/) - Industry data
- [A.M. Best Ratings](https://www.ambest.com/) - Insurer financial strength
- [NCCI](https://www.ncci.com/) - Workers' comp statistical data
- [Physician Insurers Association of America](https://www.thepiaa.org/) - Med mal data sharing

---

**Document Version:** 1.0
**Last Updated:** 2026-02-05
**Maintained By:** ALI AI Team

**Change Log:**
- v1.0 (2026-02-05): Initial insurance domain skill covering P&C fundamentals, medical malpractice, claims lifecycle, underwriting, reinsurance, financial reporting, data concepts, metrics, and common data quality issues
