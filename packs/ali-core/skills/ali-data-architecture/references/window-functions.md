# ali-data-architecture - Window Functions Reference

Complete SQL examples for analytic window functions.

---

## ROW_NUMBER - Deduplication

Remove duplicates by keeping only the latest version of each record.

```sql
-- Pattern: ROW_NUMBER() OVER (PARTITION BY key ORDER BY timestamp DESC)
SELECT *
FROM (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY claim_id  -- Group by unique identifier
            ORDER BY updated_at DESC  -- Latest first
        ) AS rn
    FROM claims
)
WHERE rn = 1;  -- Keep only latest version

-- Alternative: Qualify clause (Snowflake)
SELECT *
FROM claims
QUALIFY ROW_NUMBER() OVER (PARTITION BY claim_id ORDER BY updated_at DESC) = 1;
```

**Common use cases:**
- Deduplication in staging tables
- Latest record selection
- Top N per group (use rn <= N)

---

## RANK / DENSE_RANK - Ranking with Ties

Assign ranks with tie handling.

```sql
-- RANK: Gaps after ties (1, 2, 2, 4)
SELECT
    provider_id,
    provider_name,
    total_claims,
    RANK() OVER (ORDER BY total_claims DESC) AS rank
FROM provider_summary;

-- DENSE_RANK: No gaps after ties (1, 2, 2, 3)
SELECT
    provider_id,
    provider_name,
    total_claims,
    DENSE_RANK() OVER (ORDER BY total_claims DESC) AS dense_rank
FROM provider_summary;

-- Rank within partition
SELECT
    specialty,
    provider_id,
    total_claims,
    RANK() OVER (PARTITION BY specialty ORDER BY total_claims DESC) AS rank_in_specialty
FROM provider_summary;
```

**Difference:**
- **RANK**: `1, 2, 2, 4` (skips 3 after tie)
- **DENSE_RANK**: `1, 2, 2, 3` (no gaps)
- **ROW_NUMBER**: `1, 2, 3, 4` (unique, arbitrary order for ties)

---

## SUM - Running Totals

Calculate cumulative sums.

```sql
-- Running total for each client
SELECT
    client_id,
    payment_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY client_id
        ORDER BY payment_date
        ROWS UNBOUNDED PRECEDING  -- From start to current row
    ) AS running_total
FROM payments
ORDER BY client_id, payment_date;

-- Running total with explicit frame
SELECT
    client_id,
    payment_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY client_id
        ORDER BY payment_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM payments;
```

**Frame clauses:**
- `UNBOUNDED PRECEDING`: From first row in partition
- `CURRENT ROW`: Up to current row
- `N PRECEDING`: N rows before current
- `N FOLLOWING`: N rows after current

---

## AVG - Moving Average

Calculate rolling averages.

```sql
-- 7-day moving average
SELECT
    report_date,
    daily_count,
    AVG(daily_count) OVER (
        ORDER BY report_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW  -- 7 days total
    ) AS seven_day_avg
FROM daily_metrics
ORDER BY report_date;

-- 30-day moving average
SELECT
    report_date,
    daily_revenue,
    AVG(daily_revenue) OVER (
        ORDER BY report_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW  -- 30 days total
    ) AS thirty_day_avg
FROM daily_revenue;

-- Centered moving average (3 days)
SELECT
    report_date,
    daily_count,
    AVG(daily_count) OVER (
        ORDER BY report_date
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING  -- Yesterday, today, tomorrow
    ) AS three_day_centered_avg
FROM daily_metrics;
```

---

## LAG / LEAD - Compare with Previous/Next Rows

Access values from adjacent rows.

```sql
-- Year-over-year comparison
SELECT
    client_id,
    tax_year,
    refund_amount,
    LAG(refund_amount) OVER (
        PARTITION BY client_id
        ORDER BY tax_year
    ) AS prior_year_refund,
    refund_amount - LAG(refund_amount) OVER (
        PARTITION BY client_id
        ORDER BY tax_year
    ) AS year_over_year_change,
    CASE
        WHEN LAG(refund_amount) OVER (PARTITION BY client_id ORDER BY tax_year) > 0
        THEN ((refund_amount - LAG(refund_amount) OVER (PARTITION BY client_id ORDER BY tax_year))
              / LAG(refund_amount) OVER (PARTITION BY client_id ORDER BY tax_year)) * 100
        ELSE NULL
    END AS yoy_change_pct
FROM tax_returns
ORDER BY client_id, tax_year;

-- Next value (LEAD)
SELECT
    claim_id,
    service_date,
    total_charge,
    LEAD(service_date) OVER (
        PARTITION BY patient_id
        ORDER BY service_date
    ) AS next_service_date,
    DATEDIFF(day,
        service_date,
        LEAD(service_date) OVER (PARTITION BY patient_id ORDER BY service_date)
    ) AS days_until_next_service
FROM claims;

-- LAG with offset and default
SELECT
    claim_id,
    service_date,
    LAG(service_date, 2, '1900-01-01') OVER (  -- 2 rows back, default if NULL
        PARTITION BY patient_id
        ORDER BY service_date
    ) AS service_date_two_ago
FROM claims;
```

---

## FIRST_VALUE / LAST_VALUE - Boundaries

Get first or last value in window.

```sql
-- First payment date for each claim
SELECT DISTINCT
    claim_id,
    FIRST_VALUE(payment_date) OVER (
        PARTITION BY claim_id
        ORDER BY payment_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_payment_date,
    LAST_VALUE(payment_date) OVER (
        PARTITION BY claim_id
        ORDER BY payment_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_payment_date
FROM payments;

-- Compare current value to baseline (first value)
SELECT
    provider_id,
    month,
    monthly_revenue,
    FIRST_VALUE(monthly_revenue) OVER (
        PARTITION BY provider_id
        ORDER BY month
    ) AS baseline_revenue,
    monthly_revenue - FIRST_VALUE(monthly_revenue) OVER (
        PARTITION BY provider_id
        ORDER BY month
    ) AS revenue_change_from_baseline
FROM monthly_provider_revenue;
```

---

## NTH_VALUE - Specific Position

Get value at specific position in window.

```sql
-- Second highest payment amount for each claim
SELECT DISTINCT
    claim_id,
    NTH_VALUE(payment_amount, 2) OVER (
        PARTITION BY claim_id
        ORDER BY payment_amount DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_highest_payment
FROM payments;
```

---

## PERCENT_RANK - Relative Rank

Calculate percentile rank (0 to 1).

```sql
-- Percentile rank of providers by claim volume
SELECT
    provider_id,
    provider_name,
    total_claims,
    PERCENT_RANK() OVER (ORDER BY total_claims DESC) AS percentile_rank,
    ROUND(PERCENT_RANK() OVER (ORDER BY total_claims DESC) * 100, 2) AS percentile
FROM provider_summary;

-- Quartile assignment
SELECT
    provider_id,
    total_claims,
    NTILE(4) OVER (ORDER BY total_claims DESC) AS quartile  -- 1=top 25%, 4=bottom 25%
FROM provider_summary;
```

---

## CUME_DIST - Cumulative Distribution

Calculate cumulative distribution (percentage of rows <= current row).

```sql
-- Cumulative distribution of claim amounts
SELECT
    claim_id,
    total_charge,
    CUME_DIST() OVER (ORDER BY total_charge) AS cumulative_dist,
    ROUND(CUME_DIST() OVER (ORDER BY total_charge) * 100, 2) AS cumulative_pct
FROM claims;
```

---

## Complex Examples

### Gap and Island Detection

Find consecutive sequences.

```sql
-- Identify gaps in claim sequence
WITH numbered_claims AS (
    SELECT
        claim_id,
        service_date,
        ROW_NUMBER() OVER (ORDER BY service_date) AS rn,
        DATE_PART(day, service_date) - ROW_NUMBER() OVER (ORDER BY service_date) AS grp
    FROM claims
    WHERE patient_id = 'P001'
)
SELECT
    MIN(service_date) AS island_start,
    MAX(service_date) AS island_end,
    COUNT(*) AS consecutive_days
FROM numbered_claims
GROUP BY grp
ORDER BY island_start;
```

### Session Identification

Group events into sessions based on time gaps.

```sql
-- Identify user sessions (gap > 30 minutes = new session)
WITH session_starts AS (
    SELECT
        user_id,
        event_timestamp,
        CASE
            WHEN LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp) IS NULL
                THEN 1
            WHEN DATEDIFF(minute,
                    LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp),
                    event_timestamp
                 ) > 30
                THEN 1
            ELSE 0
        END AS is_session_start
    FROM user_events
)
SELECT
    user_id,
    event_timestamp,
    SUM(is_session_start) OVER (
        PARTITION BY user_id
        ORDER BY event_timestamp
        ROWS UNBOUNDED PRECEDING
    ) AS session_id
FROM session_starts;
```

### Conditional Aggregation

Aggregate with conditions.

```sql
-- Count events meeting criteria within window
SELECT
    claim_id,
    service_date,
    COUNT(CASE WHEN diagnosis_code LIKE 'E%' THEN 1 END) OVER (
        PARTITION BY patient_id
        ORDER BY service_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW  -- Last 7 claims
    ) AS diabetes_diagnoses_last_7
FROM claims;
```

---

## Frame Specifications

### ROWS vs RANGE

```sql
-- ROWS: Physical offset (count rows)
SELECT
    service_date,
    amount,
    SUM(amount) OVER (
        ORDER BY service_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW  -- Exactly 3 rows
    ) AS sum_last_3_rows
FROM claims;

-- RANGE: Logical offset (same ORDER BY values)
SELECT
    service_date,
    amount,
    SUM(amount) OVER (
        ORDER BY service_date
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW  -- All rows in last 7 days
    ) AS sum_last_7_days
FROM claims;
```

### Common Frame Patterns

| Pattern | Frame Clause | Use Case |
|---------|--------------|----------|
| **All rows** | `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | Overall aggregates |
| **Running total** | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | Cumulative sum |
| **Moving average** | `ROWS BETWEEN N PRECEDING AND CURRENT ROW` | N-period average |
| **Centered window** | `ROWS BETWEEN N PRECEDING AND N FOLLOWING` | Smoothing |
| **Until next** | `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` | Forward-looking |

---

## Performance Tips

1. **Avoid multiple OVER clauses**: Define window once, reuse

```sql
-- BAD: Define window twice
SELECT
    claim_id,
    SUM(amount) OVER (PARTITION BY patient_id ORDER BY service_date),
    AVG(amount) OVER (PARTITION BY patient_id ORDER BY service_date)
FROM claims;

-- GOOD: Define window once
SELECT
    claim_id,
    SUM(amount) OVER w,
    AVG(amount) OVER w
FROM claims
WINDOW w AS (PARTITION BY patient_id ORDER BY service_date);
```

2. **Order by indexed columns**: Speeds up window sorting

3. **Use QUALIFY instead of subquery** (Snowflake):

```sql
-- Instead of:
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (...) AS rn
    FROM table
) WHERE rn = 1;

-- Use:
SELECT * FROM table
QUALIFY ROW_NUMBER() OVER (...) = 1;
```

---

## Summary

- **ROW_NUMBER**: Deduplication, top N per group
- **RANK/DENSE_RANK**: Ranking with tie handling
- **SUM/AVG**: Running totals, moving averages
- **LAG/LEAD**: Period-over-period comparisons
- **FIRST_VALUE/LAST_VALUE**: Boundary values in window
- **Frames**: ROWS (physical) vs RANGE (logical)
