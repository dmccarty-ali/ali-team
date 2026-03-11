# ali-wherescape - Object Types Reference

This reference provides detailed implementation examples with generated SQL for all WhereScape object types.

---

## Load Tables

**Purpose:** Extract data from source systems into staging

```
Source System → Load Table → Stage Table
```

**Configuration:**
- Source connection (ODBC/JDBC, file path, API endpoint)
- Extraction method (full, incremental, CDC)
- Column mappings
- Source query or table name

**Generated Code:**
- INSERT statement (full load)
- MERGE/UPSERT (incremental)
- DELETE + INSERT (delete-and-reload)

**Example:**
```sql
-- Generated Load Table code (Snowflake target)
TRUNCATE TABLE LOAD_CUSTOMER;

INSERT INTO LOAD_CUSTOMER (
    customer_id,
    customer_name,
    email,
    created_date,
    load_timestamp
)
SELECT
    id AS customer_id,
    name AS customer_name,
    email_address AS email,
    created_at AS created_date,
    CURRENT_TIMESTAMP AS load_timestamp
FROM source_db.customers
WHERE updated_at > :last_load_timestamp;
```

---

## Stage Tables

**Purpose:** Cleanse, validate, and prepare data for warehouse

```
Load Table → Stage Table → Dimension/Fact/Vault
```

**Configuration:**
- Source table (Load table or other Stage table)
- Transformation logic (derived columns, lookups)
- Data quality rules
- Slowly changing dimension type (1, 2, 3)

**Generated Code:**
- MERGE statement for SCD Type 2
- UPDATE for SCD Type 1
- Validation checks
- Hash key generation (for Data Vault)

**Example (SCD Type 2):**
```sql
-- Generated Stage Table code (SCD Type 2)
MERGE INTO STG_CUSTOMER tgt
USING (
    SELECT
        customer_id,
        customer_name,
        email,
        MD5(customer_id) AS customer_hkey,
        MD5(customer_name || '|' || email) AS customer_hdiff,
        CURRENT_TIMESTAMP AS load_timestamp,
        '9999-12-31' AS valid_to,
        TRUE AS is_current
    FROM LOAD_CUSTOMER
) src
ON tgt.customer_hkey = src.customer_hkey
   AND tgt.is_current = TRUE
WHEN MATCHED AND tgt.customer_hdiff <> src.customer_hdiff THEN
    UPDATE SET
        tgt.valid_to = CURRENT_DATE - 1,
        tgt.is_current = FALSE
WHEN NOT MATCHED THEN
    INSERT (
        customer_hkey,
        customer_id,
        customer_name,
        email,
        customer_hdiff,
        valid_from,
        valid_to,
        is_current,
        load_timestamp
    )
    VALUES (
        src.customer_hkey,
        src.customer_id,
        src.customer_name,
        src.email,
        src.customer_hdiff,
        CURRENT_DATE,
        '9999-12-31',
        TRUE,
        src.load_timestamp
    );
```

---

## Hub (Data Vault)

**Purpose:** Store unique business keys

```
Stage Table → Hub
```

**Configuration:**
- Business key columns
- Source table
- Load date tracking

**Generated Code:**
- Hash key calculation (MD5 or SHA-256)
- INSERT with deduplication

**Example:**
```sql
-- Generated Hub code
INSERT INTO HUB_CUSTOMER (
    customer_hkey,
    customer_id,
    load_date,
    record_source
)
SELECT DISTINCT
    MD5(customer_id) AS customer_hkey,
    customer_id,
    CURRENT_DATE AS load_date,
    'CRM_SYSTEM' AS record_source
FROM STG_CUSTOMER src
WHERE NOT EXISTS (
    SELECT 1 FROM HUB_CUSTOMER hub
    WHERE hub.customer_hkey = MD5(src.customer_id)
);
```

---

## Link (Data Vault)

**Purpose:** Store relationships between business entities

```
Hub A + Hub B → Link
```

**Configuration:**
- Hub references (parent hubs)
- Relationship keys
- Load date tracking

**Generated Code:**
- Composite hash key from parent hubs
- INSERT with deduplication

**Example:**
```sql
-- Generated Link code
INSERT INTO LINK_CUSTOMER_ORDER (
    customer_order_hkey,
    customer_hkey,
    order_hkey,
    load_date,
    record_source
)
SELECT DISTINCT
    MD5(customer_hkey || '|' || order_hkey) AS customer_order_hkey,
    customer_hkey,
    order_hkey,
    CURRENT_DATE AS load_date,
    'ORDER_SYSTEM' AS record_source
FROM STG_ORDER stg
JOIN HUB_CUSTOMER hc ON hc.customer_id = stg.customer_id
JOIN HUB_ORDER ho ON ho.order_id = stg.order_id
WHERE NOT EXISTS (
    SELECT 1 FROM LINK_CUSTOMER_ORDER link
    WHERE link.customer_order_hkey = MD5(hc.customer_hkey || '|' || ho.order_hkey)
);
```

---

## Satellite (Data Vault)

**Purpose:** Store descriptive attributes and history

```
Hub/Link → Satellite
```

**Configuration:**
- Parent hub or link
- Attribute columns
- Hash diff for change detection
- Load date tracking

**Generated Code:**
- MERGE statement with hash diff comparison
- SCD Type 2 pattern

**Example:**
```sql
-- Generated Satellite code
INSERT INTO SAT_CUSTOMER (
    customer_hkey,
    load_date,
    load_end_date,
    customer_name,
    email,
    phone,
    address,
    hash_diff,
    record_source
)
SELECT
    hc.customer_hkey,
    CURRENT_DATE AS load_date,
    '9999-12-31' AS load_end_date,
    stg.customer_name,
    stg.email,
    stg.phone,
    stg.address,
    MD5(stg.customer_name || '|' || stg.email || '|' || stg.phone || '|' || stg.address) AS hash_diff,
    'CRM_SYSTEM' AS record_source
FROM STG_CUSTOMER stg
JOIN HUB_CUSTOMER hc ON hc.customer_id = stg.customer_id
WHERE NOT EXISTS (
    SELECT 1 FROM SAT_CUSTOMER sat
    WHERE sat.customer_hkey = hc.customer_hkey
      AND sat.load_end_date = '9999-12-31'
      AND sat.hash_diff = MD5(stg.customer_name || '|' || stg.email || '|' || stg.phone || '|' || stg.address)
);

-- Close out previous records
UPDATE SAT_CUSTOMER
SET load_end_date = CURRENT_DATE - 1
WHERE customer_hkey IN (SELECT customer_hkey FROM STG_CUSTOMER)
  AND load_end_date = '9999-12-31'
  AND hash_diff NOT IN (
      SELECT MD5(customer_name || '|' || email || '|' || phone || '|' || address)
      FROM STG_CUSTOMER
  );
```

---

## Dimension Tables

**Purpose:** Business-ready descriptive attributes

```
Stage/Satellite → Dimension
```

**Configuration:**
- Source tables (Stage, Satellite, or other Dimensions)
- Surrogate key generation
- SCD type (1, 2, 3)
- Attribute mappings

**Generated Code:**
- MERGE statement (SCD Type 2)
- Sequence-based surrogate keys

**Example:**
```sql
-- Generated Dimension code (SCD Type 2)
MERGE INTO DIM_CUSTOMER tgt
USING (
    SELECT
        customer_id,
        customer_name,
        email,
        phone,
        address
    FROM STG_CUSTOMER
) src
ON tgt.customer_id = src.customer_id AND tgt.is_current = TRUE
WHEN MATCHED AND (
    tgt.customer_name <> src.customer_name OR
    tgt.email <> src.email OR
    tgt.phone <> src.phone OR
    tgt.address <> src.address
) THEN
    UPDATE SET
        tgt.valid_to = CURRENT_DATE - 1,
        tgt.is_current = FALSE
WHEN NOT MATCHED THEN
    INSERT (
        customer_sk,
        customer_id,
        customer_name,
        email,
        phone,
        address,
        valid_from,
        valid_to,
        is_current
    )
    VALUES (
        SEQ_DIM_CUSTOMER.NEXTVAL,
        src.customer_id,
        src.customer_name,
        src.email,
        src.phone,
        src.address,
        CURRENT_DATE,
        '9999-12-31',
        TRUE
    );
```

---

## Fact Tables

**Purpose:** Business metrics at defined grain

```
Dimensions + Stage → Fact
```

**Configuration:**
- Source tables (Stage, other Facts)
- Dimension lookups (surrogate key mappings)
- Measure columns
- Grain definition

**Generated Code:**
- INSERT with dimension key lookups
- Pre-aggregation (if needed)

**Example:**
```sql
-- Generated Fact code
INSERT INTO FACT_ORDER (
    order_sk,
    customer_sk,
    product_sk,
    order_date_sk,
    order_id,
    quantity,
    unit_price,
    total_amount
)
SELECT
    SEQ_FACT_ORDER.NEXTVAL AS order_sk,
    dc.customer_sk,
    dp.product_sk,
    dd.date_sk AS order_date_sk,
    stg.order_id,
    stg.quantity,
    stg.unit_price,
    stg.quantity * stg.unit_price AS total_amount
FROM STG_ORDER stg
JOIN DIM_CUSTOMER dc ON dc.customer_id = stg.customer_id AND dc.is_current = TRUE
JOIN DIM_PRODUCT dp ON dp.product_id = stg.product_id AND dp.is_current = TRUE
JOIN DIM_DATE dd ON dd.full_date = stg.order_date
WHERE NOT EXISTS (
    SELECT 1 FROM FACT_ORDER fo
    WHERE fo.order_id = stg.order_id
);
```

---

## Aggregate Tables

**Purpose:** Pre-calculated summaries for performance

```
Fact Table → Aggregate
```

**Configuration:**
- Source fact table
- Grouping dimensions
- Aggregation functions (SUM, AVG, COUNT, MIN, MAX)

**Generated Code:**
- GROUP BY with aggregation functions
- DELETE-and-reload or incremental MERGE

**Example:**
```sql
-- Generated Aggregate code
TRUNCATE TABLE AGG_DAILY_SALES;

INSERT INTO AGG_DAILY_SALES (
    order_date_sk,
    product_sk,
    total_orders,
    total_quantity,
    total_revenue,
    avg_order_value
)
SELECT
    order_date_sk,
    product_sk,
    COUNT(DISTINCT order_id) AS total_orders,
    SUM(quantity) AS total_quantity,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value
FROM FACT_ORDER
GROUP BY order_date_sk, product_sk;
```

---

**Document Version:** 1.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
