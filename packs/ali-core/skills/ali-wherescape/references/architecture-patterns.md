# ali-wherescape - Architecture Patterns Reference

This reference provides detailed guidance on mapping WhereScape to architectural patterns like Medallion and Data Vault automation.

---

## Medallion Architecture Mapping

WhereScape objects map to medallion layers:

| Medallion Layer | WhereScape Object | Purpose |
|-----------------|-------------------|---------|
| **Bronze (LAKE)** | Load Table | Raw ingestion, append-only |
| **Silver (STG)** | Stage Table | Cleansed, validated, deduplicated |
| **Gold (EDW)** | Dimension, Fact, Aggregate | Business-ready, star schema |

### Bronze Layer (Load Tables)

```
Source System → WhereScape Load Table → Bronze/LAKE Layer
```

**Configuration:**
- Minimal transformation
- Preserve source format
- Add audit columns (load_timestamp, source_file_id)
- Append-only pattern

### Silver Layer (Stage Tables)

```
Bronze/LAKE → WhereScape Stage Table → Silver/STG Layer
```

**Configuration:**
- Data quality validation
- Business typing
- Deduplication (hash keys)
- SCD Type 2 for history
- Hash diff for change detection

### Gold Layer (Dimension/Fact)

```
Silver/STG → WhereScape Dimension/Fact → Gold/EDW Layer
```

**Configuration:**
- Star schema design
- Surrogate keys
- Pre-joined, pre-aggregated
- Optimized for BI queries

---

## Data Vault Automation

WhereScape accelerates Data Vault 2.0 implementation through templates.

### Hub Generation

**Metadata configuration:**
- Table type: Hub
- Business key columns: customer_id
- Load date tracking: enabled
- Record source: CRM_SYSTEM

**Generated objects:**
- HUB_CUSTOMER table
- Load procedure
- Hash key calculation (MD5)

### Link Generation

**Metadata configuration:**
- Table type: Link
- Parent hubs: HUB_CUSTOMER, HUB_ORDER
- Relationship key: customer_id + order_id
- Load date tracking: enabled

**Generated objects:**
- LINK_CUSTOMER_ORDER table
- Load procedure
- Composite hash key

### Satellite Generation

**Metadata configuration:**
- Table type: Satellite
- Parent hub: HUB_CUSTOMER
- Attribute columns: customer_name, email, phone, address
- Hash diff: enabled
- Load date tracking: enabled

**Generated objects:**
- SAT_CUSTOMER table
- Load procedure
- Hash diff calculation
- SCD Type 2 logic

### Business Vault

Business Vault objects (derived from Raw Vault):

| Object Type | WhereScape Equivalent | Purpose |
|-------------|----------------------|---------|
| **Calculated Satellite** | Stage Table | Derived attributes |
| **Aggregated Fact** | Aggregate Table | Pre-calculated metrics |
| **Reference Table** | Dimension Table | Business-ready lookups |

---

## Performance Optimization

### Parallel Loading

**WhereScape supports parallel execution:**
- Configure job parallelism (number of concurrent jobs)
- Partition large tables into chunks
- Use database-specific parallel hints

**Example (Snowflake):**
```sql
-- Generated Load Table with parallel hint
COPY INTO LOAD_CUSTOMER
FROM @stage_area/customer_*.csv
FILE_FORMAT = (TYPE = 'CSV')
PARALLEL = 8;  -- WhereScape configuration sets parallel degree
```

### Incremental Patterns

**Watermark-based incremental:**
```sql
-- Generated Load Table (incremental)
INSERT INTO LOAD_CUSTOMER
SELECT *
FROM source_db.customers
WHERE updated_at > (
    SELECT MAX(load_timestamp) FROM dss_control.load_watermarks
    WHERE table_name = 'LOAD_CUSTOMER'
);

-- Update watermark
UPDATE dss_control.load_watermarks
SET last_load_timestamp = (SELECT MAX(updated_at) FROM LOAD_CUSTOMER)
WHERE table_name = 'LOAD_CUSTOMER';
```

**CDC-based incremental:**
```sql
-- Generated Load Table (CDC)
INSERT INTO LOAD_CUSTOMER
SELECT *
FROM source_db.customer_cdc_stream
WHERE change_timestamp > :last_cdc_timestamp;
```

### Partition Management

**WhereScape can generate partition-aware SQL:**
```sql
-- Generated Stage Table with partition pruning
MERGE INTO STG_ORDER_PARTITIONED tgt
USING LOAD_ORDER src
ON tgt.order_id = src.order_id
   AND tgt.order_date = src.order_date  -- Partition key
WHEN MATCHED THEN UPDATE ...
WHEN NOT MATCHED THEN INSERT ...;
```

---

**Document Version:** 1.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
