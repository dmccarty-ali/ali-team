# ali-data-architecture - Data Lineage Reference

Column-level lineage tracking for transparency and compliance.

---

## Column-Level Lineage

Track transformation from source to target at column level.

```yaml
# metadata/lineage.yaml
EDW_MEDEDI.FACT_CLAIM.total_paid:
  source: STG_MEDEDI.PAYMENT.paid_amount
  transformation: SUM(paid_amount) GROUP BY claim_id
  business_rule: "Sum of all payments for claim"

STG_MEDEDI.PAYMENT.paid_amount:
  source: LAKE_MEDEDI.EDI_835_RAW.CLP03
  transformation: CAST to NUMBER(12,2)
  business_rule: "Payment amount from 835 CLP segment"
```

---

## Audit Columns

Standard audit columns on all tables.

```sql
-- Required on all staging and EDW tables
load_timestamp      TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
loaded_by           VARCHAR(100) DEFAULT CURRENT_USER,
source_file_id      VARCHAR(100),    -- Traceability to LAKE layer
source_system       VARCHAR(50),     -- Which system generated this
etl_batch_id        VARCHAR(100)     -- Batch identifier for reconciliation
```

---

## Benefits

- **Compliance**: Audit trail for regulations
- **Debugging**: Trace bad data to source
- **Impact analysis**: Which downstream tables affected by source change
- **Trust**: Transparency in transformation logic
