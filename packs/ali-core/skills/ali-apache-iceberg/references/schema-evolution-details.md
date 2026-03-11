# ali-apache-iceberg - Schema Evolution Details Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg schema evolution examples" or "I need detailed ALTER TABLE syntax"

---

## Add Column Examples

### Basic Add Column

```sql
-- Add new column (default NULL for existing data)
ALTER TABLE my_catalog.bronze_layer.customers
ADD COLUMN email STRING;

-- Add with comment
ALTER TABLE customers
ADD COLUMN phone_number STRING COMMENT 'Contact phone';

-- Old data: email = NULL (no rewrite needed)
-- New data: email can be populated
```

### Add Column with Position

```sql
-- Add column at specific position
ALTER TABLE customers
ADD COLUMN middle_name STRING AFTER first_name;

-- Add as first column
ALTER TABLE customers
ADD COLUMN row_version BIGINT FIRST;
```

### Add Nested Column

```sql
-- Add field to struct
ALTER TABLE orders
ADD COLUMN shipping_address.zip_code STRING;

-- Before: shipping_address: STRUCT<street: STRING, city: STRING>
-- After:  shipping_address: STRUCT<street: STRING, city: STRING, zip_code: STRING>
```

---

## Drop Column Examples

### Basic Drop Column

```sql
-- Drop column (metadata-only operation)
ALTER TABLE customers
DROP COLUMN deprecated_field;

-- Data files not rewritten
-- Column ignored on reads
-- Can still time-travel to read old data with this column
```

### Drop Nested Column

```sql
-- Drop field from struct
ALTER TABLE orders
DROP COLUMN shipping_address.apartment_number;
```

---

## Rename Column Examples

### Basic Rename

```sql
-- Rename without rewriting data
ALTER TABLE customers
RENAME COLUMN customer_name TO full_name;

-- Readers using old schema still work (column ID unchanged)
-- Iceberg tracks column by ID, not name
```

### Rename Nested Column

```sql
-- Rename field in struct
ALTER TABLE orders
RENAME COLUMN shipping_address.street TO street_address;
```

---

## Type Promotion (Safe Widening)

### Integer Widening

```sql
-- Widen integer type
ALTER TABLE orders
ALTER COLUMN quantity TYPE BIGINT;

-- Before: quantity INT (32-bit)
-- After:  quantity BIGINT (64-bit)
-- Old data readable as BIGINT
```

### Float to Double

```sql
-- Promote float to double precision
ALTER TABLE measurements
ALTER COLUMN temperature TYPE DOUBLE;

-- Before: temperature FLOAT (32-bit)
-- After:  temperature DOUBLE (64-bit)
```

### Decimal Precision Increase

```sql
-- Increase decimal precision
ALTER TABLE transactions
ALTER COLUMN amount TYPE DECIMAL(20, 2);

-- Before: amount DECIMAL(10, 2)
-- After:  amount DECIMAL(20, 2)
-- More precision for large values
```

---

## Type Compatibility Table

| From Type | To Type | Safe? | Notes |
|-----------|---------|-------|-------|
| int | bigint | ✅ Yes | Common widening |
| bigint | int | ❌ No | Data loss risk |
| float | double | ✅ Yes | Precision increase |
| double | float | ❌ No | Precision loss |
| decimal(10,2) | decimal(20,2) | ✅ Yes | Scale must match |
| decimal(10,2) | decimal(10,4) | ❌ No | Scale change unsafe |
| string | binary | ❌ No | Encoding change |
| date | timestamp | ✅ Yes | Add time component |
| timestamp | date | ❌ No | Lose time information |

**Rule**: Widening is safe, narrowing is not.

---

## Reorder Columns

```sql
-- Move column to first position
ALTER TABLE customers
ALTER COLUMN customer_id FIRST;

-- Move column after another
ALTER TABLE customers
ALTER COLUMN email AFTER phone_number;

-- Physical layout unchanged (Parquet stores by name)
-- Logical schema reordered
```

---

## Update Column Comments

```sql
-- Add or update comment
ALTER TABLE customers
ALTER COLUMN email COMMENT 'Primary email address for notifications';

-- Metadata-only operation
```

---

## Complex Schema Evolution

### Evolve Struct Type

```sql
-- Initial schema
CREATE TABLE orders (
    order_id BIGINT,
    shipping STRUCT<
        street: STRING,
        city: STRING
    >
)
USING iceberg;

-- Add field to struct
ALTER TABLE orders
ADD COLUMN shipping.state STRING;

-- Add another struct field
ALTER TABLE orders
ADD COLUMN shipping.zip_code STRING;

-- Result: shipping STRUCT<street, city, state, zip_code>
```

### Evolve Array Element Type

```sql
-- Initial: array of ints
CREATE TABLE events (
    event_id BIGINT,
    related_ids ARRAY<INT>
)
USING iceberg;

-- Widen array element type
ALTER TABLE events
ALTER COLUMN related_ids TYPE ARRAY<BIGINT>;

-- Old data (INT array) readable as BIGINT array
```

### Evolve Map Value Type

```sql
-- Initial: map with string values
CREATE TABLE metadata (
    object_id BIGINT,
    properties MAP<STRING, STRING>
)
USING iceberg;

-- Cannot change map value type (not supported)
-- Workaround: add new column
ALTER TABLE metadata
ADD COLUMN properties_v2 MAP<STRING, STRUCT<value: STRING, type: STRING>>;
```

---

## Schema Evolution Limitations

### Unsupported Operations

| Operation | Supported? | Workaround |
|-----------|-----------|------------|
| Change column type (narrowing) | ❌ No | Add new column, migrate data |
| Change partition spec column type | ❌ No | Partition evolution instead |
| Rename partition column | ❌ No | Drop and add partition field |
| Change map key type | ❌ No | Create new column |
| Change map value type (narrowing) | ❌ No | Create new column |
| Reorder struct fields | ❌ No | Struct field order is fixed |

---

## Schema Compatibility Across Engines

### Spark

```python
# Spark reads Iceberg schema from metadata
df = spark.read.table("my_catalog.bronze_layer.orders")

# Schema evolution applied automatically
# Old readers see NULL for new columns
```

### Flink

```java
// Flink SQL reads Iceberg schema
tableEnv.executeSql("SELECT * FROM orders");

// Schema changes reflected immediately
// No cache invalidation needed
```

### Trino/Starburst

```sql
-- Trino reads latest schema from Iceberg metadata
SELECT * FROM iceberg.bronze_layer.orders;

-- Schema changes visible immediately
-- DESCRIBE TABLE shows current schema
```

---

## Time Travel with Schema Evolution

```sql
-- Query old schema before column added
SELECT * FROM orders
FOR SYSTEM_TIME AS OF '2026-01-01';

-- Result: old_column visible, new_column not present

-- Query current schema
SELECT * FROM orders;

-- Result: new_column present (NULL for old rows)
```

**Key insight**: Iceberg tracks schema per snapshot. Time travel restores both data AND schema.

---

## Migration Pattern: Column Replacement

When you need to "change" a column type unsafely:

```sql
-- Step 1: Add new column with desired type
ALTER TABLE orders
ADD COLUMN amount_v2 DECIMAL(20, 4);

-- Step 2: Backfill old data
UPDATE orders
SET amount_v2 = amount
WHERE amount_v2 IS NULL;

-- Step 3: Drop old column (after confirming migration)
ALTER TABLE orders
DROP COLUMN amount;

-- Step 4: Rename new column
ALTER TABLE orders
RENAME COLUMN amount_v2 TO amount;
```

---

## Best Practices

1. **Always widen, never narrow**: Use type promotion (int → bigint), not demotion
2. **Test with time travel**: Query old snapshots to verify schema compatibility
3. **Add columns at end**: Simplifies schema evolution tracking
4. **Use comments**: Document why columns were added/removed
5. **Version structs carefully**: Struct evolution is limited, plan ahead

---

## References

- [Iceberg Schema Evolution](https://iceberg.apache.org/docs/latest/evolution/)
- [Schema Evolution Spec](https://iceberg.apache.org/spec/#schema-evolution)
- [Type Promotion Rules](https://iceberg.apache.org/docs/latest/schemas/)
