# ali-tax-practice-domain - AI Analysis Reference

This reference contains detailed AI integration patterns, batch processing, prompt tracking, and prior year variance analysis specifications.

---

## AI Integration (Bedrock/Claude)

### AI-Powered Operations

| Operation | Model | Purpose |
|-----------|-------|---------|
| **Classification** | Claude | Document type identification |
| **Extraction** | Claude | Field extraction from documents |
| **Analysis** | Claude | Prior year comparison, missing docs |
| **Q&A** | Claude | Client questions about return |

---

## Batch Processing (Cost Optimization)

### Batch Job Structure

```python
class BatchJob:
    batch_id: str
    job_type: str          # "extraction", "analysis"
    scheduled_at: datetime  # 2 AM for off-peak pricing
    completed_at: datetime
    input_token_count: int
    output_token_count: int
    estimated_cost: Decimal
```

### Batch Processing Strategy

- Schedule batch jobs for 2 AM (off-peak pricing)
- Group similar operations (all extractions, all analyses)
- Track token usage for cost monitoring
- Estimate costs before execution

---

## Prompt Execution Tracking

```python
class PromptExecution:
    prompt_type: str
    input_tokens: int
    output_tokens: int
    response_time_ms: int
    cost_cents: int
    model_id: str
```

### Tracking Purposes

- Monitor AI usage and costs
- Identify slow prompts for optimization
- Track model performance over time
- Calculate per-client AI costs

---

## Prior Year Variance Analysis

### Variance Severity Levels

| Severity | Threshold | Action |
|----------|-----------|--------|
| **Low** | <10% change | Informational only |
| **Medium** | 10-30% change | Flag for preparer review |
| **High** | >30% change | Require explanation before filing |

### Variance Detection

```python
class PriorYearComparison:
    category: str           # "wages", "dividends", "deductions"
    prior_year_amount: Decimal
    current_year_amount: Decimal
    variance_percent: float
    variance_flag: VarianceFlag
    explanation: str        # Required if high severity
    reviewed_by: str
    reviewed_at: datetime
```

### Variance Categories Tracked

- Wages (W-2 income)
- Self-employment income
- Investment income (dividends, interest, capital gains)
- Retirement distributions
- Deductions (mortgage interest, property tax, charitable)
- Credits (child tax credit, education credits)

---

## Life Events (Auto-Detected)

The system can automatically detect significant life events that may explain variances:

- **Marriage / Divorce** - Filing status change
- **New child / dependent** - New dependents claimed
- **Home purchase / sale** - Mortgage interest, capital gains
- **Job change** - W-2 employer change, income variance
- **Retirement** - Retirement distributions, pension income
- **State relocation** - State tax filing change

### Detection Methods

- Compare prior year vs current year data
- Flag significant changes
- Suggest preparer questions for client
- Document explanation in return workpapers
