# ali-snowflake-data-engineer - Semi-Structured Data Reference

## Deep JSON Extraction

```sql
-- Deep JSON extraction
SELECT
    raw:order_id::INT as order_id,
    raw:customer:name::STRING as customer_name,
    raw:customer:address:city::STRING as city,
    items.value:product_id::INT as product_id,
    items.value:quantity::INT as quantity,
    items.value:price::FLOAT as price
FROM orders_raw,
LATERAL FLATTEN(input => raw:items) items;
```

## Recursive Flattening

```sql
-- Recursive flattening for nested arrays
WITH RECURSIVE flattened AS (
    SELECT
        raw:id::INT as id,
        raw:name::STRING as name,
        NULL::INT as parent_id,
        raw:children as children,
        0 as depth
    FROM hierarchy_raw

    UNION ALL

    SELECT
        c.value:id::INT,
        c.value:name::STRING,
        f.id,
        c.value:children,
        f.depth + 1
    FROM flattened f,
    LATERAL FLATTEN(input => f.children) c
    WHERE f.children IS NOT NULL
)
SELECT id, name, parent_id, depth FROM flattened;
```

## JSON Aggregation

```sql
-- JSON aggregation
SELECT
    customer_id,
    OBJECT_CONSTRUCT(
        'name', customer_name,
        'orders', ARRAY_AGG(
            OBJECT_CONSTRUCT(
                'order_id', order_id,
                'amount', amount,
                'date', order_date
            )
        )
    ) as customer_json
FROM orders
GROUP BY customer_id, customer_name;
```
