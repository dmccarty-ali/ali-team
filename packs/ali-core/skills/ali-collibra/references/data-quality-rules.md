# ali-collibra - Data Quality Rules Reference

## Data Quality & Observability

### Quality Rule Definition

```yaml
# Data Quality Rule Example
rule_name: Email Format Validation
asset_type: Column
target_column: customer_email
rule_type: Format Validation
logic: |
  SELECT COUNT(*) AS invalid_count
  FROM customers
  WHERE email NOT REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$'

threshold:
  type: percentage
  max_invalid: 1.0  # Allow up to 1% invalid

schedule: Daily at 3 AM
alerts:
  - type: email
    recipients: [data-steward@aliunde.com]
    condition: threshold_exceeded
```

### Quality Scorecard

```yaml
# Scorecard Configuration
scorecard_name: Customer Data Quality
scope: Customer_Master table

dimensions:
  - completeness:
      weight: 30%
      rules:
        - email_not_null
        - address_not_null
        - phone_not_null

  - accuracy:
      weight: 40%
      rules:
        - email_format_valid
        - phone_format_valid
        - zip_code_valid

  - consistency:
      weight: 20%
      rules:
        - state_matches_zip
        - area_code_matches_state

  - timeliness:
      weight: 10%
      rules:
        - updated_within_90_days

overall_score_calculation: weighted_average
refresh_frequency: daily
```
