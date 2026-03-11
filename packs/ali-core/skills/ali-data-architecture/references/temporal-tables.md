# ali-data-architecture - Temporal Tables Reference

Bitemporal table design for tracking data as of any point in time.

---

## Bitemporal Table Design

Track two time dimensions: valid time (when true in reality) and transaction time (when recorded in system).

```sql
CREATE TABLE dim_customer (
    customer_sk         INTEGER PRIMARY KEY,
    customer_id         VARCHAR(50),
    name                VARCHAR(200),
    email               VARCHAR(255),
    -- Valid time (when true in business reality)
    valid_from          DATE NOT NULL,
    valid_to            DATE NOT NULL DEFAULT '9999-12-31',
    -- Transaction time (when recorded in system)
    sys_start           TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP,
    sys_end             TIMESTAMP_NTZ DEFAULT '9999-12-31 23:59:59',
    is_current          BOOLEAN DEFAULT TRUE
);
```

---

## Querying Bitemporal Tables

### As of Business Date

```sql
-- What did we know about customer on 2024-06-15?
SELECT * FROM dim_customer
WHERE customer_id = 'C001'
  AND valid_from <= '2024-06-15'
  AND valid_to > '2024-06-15';
```

### As of System Time

```sql
-- What did the system show on 2024-01-01?
SELECT * FROM dim_customer
WHERE customer_id = 'C001'
  AND sys_start <= '2024-01-01 00:00:00'
  AND sys_end > '2024-01-01 00:00:00';
```

### Current State

```sql
SELECT * FROM dim_customer
WHERE customer_id = 'C001'
  AND is_current = TRUE;
```

---

## Use Cases

- **Regulatory compliance**: Reconstruct data as of audit date
- **Late-arriving corrections**: Backdate changes without losing history
- **Temporal queries**: "What was the address on service date?"
