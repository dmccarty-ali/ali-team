# ali-data-architecture - Surrogate Key Generation Reference

Methods for generating surrogate keys in dimension tables.

---

## Sequence-Based (Auto-Increment)

```sql
-- Create sequence
CREATE SEQUENCE seq_customer_sk START = 1 INCREMENT = 1;

-- Generate surrogate key
INSERT INTO dim_customer (customer_sk, customer_id, name)
SELECT
    seq_customer_sk.NEXTVAL,
    customer_id,
    name
FROM staging_customers;
```

**Pros**: Simple, guaranteed unique
**Cons**: Not deterministic (different on each run), requires database sequence

---

## Hash-Based (Deterministic)

```sql
-- MD5 hash of natural key + valid_from
INSERT INTO dim_customer (customer_sk, customer_id, name, valid_from)
SELECT
    ABS(HASH(customer_id || '|' || valid_from)) AS customer_sk,
    customer_id,
    name,
    valid_from
FROM staging_customers;
```

**Pros**: Deterministic (same input = same key), no sequence needed
**Cons**: Tiny collision risk, harder to read

---

## UUID-Based (Globally Unique)

```sql
-- Generate UUID
INSERT INTO dim_customer (customer_sk, customer_id, name)
SELECT
    UUID_STRING() AS customer_sk,
    customer_id,
    name
FROM staging_customers;
```

**Pros**: Globally unique, no coordination needed
**Cons**: Larger storage (36 bytes), not sequential

---

## Composite Key (Natural + Version)

```sql
-- customer_id + version number
INSERT INTO dim_customer (customer_sk, customer_id, version, name)
SELECT
    customer_id || '_' || ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY valid_from) AS customer_sk,
    customer_id,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY valid_from) AS version,
    name
FROM staging_customers;
```

**Pros**: Human-readable, indicates version
**Cons**: Not purely surrogate (contains business meaning)

---

## Recommendation

- **Most cases**: Sequence-based (simple, standard)
- **Distributed systems**: UUID or hash-based (no coordination)
- **Deterministic replay**: Hash-based (reproducible)
