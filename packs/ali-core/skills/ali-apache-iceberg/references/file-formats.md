# ali-apache-iceberg - File Formats Reference

**Parent Skill**: ali-apache-iceberg
**Load with**: "Show me Iceberg file format details" or "I need Parquet/ORC/Avro configuration examples"

---

## Parquet Configuration (Recommended)

### Basic Parquet Setup

```sql
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_data STRING
)
USING iceberg
TBLPROPERTIES (
    'write.format.default' = 'parquet',
    'write.parquet.compression-codec' = 'zstd',
    'write.parquet.row-group-size-bytes' = '134217728',  -- 128 MB
    'write.parquet.page-size-bytes' = '1048576',  -- 1 MB
    'write.parquet.dict-size-bytes' = '2097152'  -- 2 MB dictionary
);
```

### Parquet Properties Explained

| Property | Default | Recommended | Purpose |
|----------|---------|------------|---------|
| `compression-codec` | gzip | zstd | Balance compression/speed |
| `row-group-size-bytes` | 128 MB | 128 MB | Columnar chunk size |
| `page-size-bytes` | 1 MB | 1 MB | Granularity for column reads |
| `dict-size-bytes` | 2 MB | 2 MB | Dictionary encoding threshold |

### Compression Codec Comparison

| Codec | Compression Ratio | Encode Speed | Decode Speed | Use Case |
|-------|------------------|--------------|--------------|----------|
| **zstd** | 3.5:1 | 500 MB/s | 1500 MB/s | **Recommended default** |
| **snappy** | 2:1 | 550 MB/s | 2000 MB/s | Low-latency queries |
| **lz4** | 2:1 | 750 MB/s | 2500 MB/s | High write throughput |
| **gzip** | 4:1 | 100 MB/s | 300 MB/s | Cold storage, archival |
| **brotli** | 4.5:1 | 50 MB/s | 400 MB/s | Maximum compression |
| **none** | 1:1 | ∞ | ∞ | Temporary/staging tables |

**Benchmark context**: Intel Xeon 2.5GHz, single-threaded

### Choosing Compression Codec

```sql
-- High write throughput (streaming ingestion)
TBLPROPERTIES ('write.parquet.compression-codec' = 'lz4');

-- Balanced performance (analytics)
TBLPROPERTIES ('write.parquet.compression-codec' = 'zstd');

-- Maximum compression (archival)
TBLPROPERTIES ('write.parquet.compression-codec' = 'gzip');

-- Low query latency (interactive dashboards)
TBLPROPERTIES ('write.parquet.compression-codec' = 'snappy');
```

### Advanced Parquet Tuning

```sql
CREATE TABLE high_performance_table (
    id BIGINT,
    data STRING
)
USING iceberg
TBLPROPERTIES (
    -- Format
    'write.format.default' = 'parquet',
    'write.parquet.compression-codec' = 'zstd',

    -- Performance tuning
    'write.parquet.bloom-filter-enabled.column.id' = 'true',
    'write.parquet.bloom-filter-max-bytes' = '1048576',

    -- Row group optimization
    'write.parquet.row-group-size-bytes' = '268435456',  -- 256 MB for large datasets

    -- Page-level optimization
    'write.parquet.page-size-bytes' = '2097152',  -- 2 MB for better compression
    'write.parquet.page-row-limit' = '20000',

    -- Dictionary encoding
    'write.parquet.dict-size-bytes' = '4194304'  -- 4 MB for high-cardinality columns
);
```

---

## ORC Configuration

### Basic ORC Setup

```sql
CREATE TABLE orders_orc (
    order_id BIGINT,
    customer_id BIGINT,
    order_data STRING
)
USING iceberg
TBLPROPERTIES (
    'write.format.default' = 'orc',
    'write.orc.compression-codec' = 'zstd',
    'write.orc.stripe-size-bytes' = '67108864',  -- 64 MB
    'write.orc.block-size-bytes' = '268435456'  -- 256 MB
);
```

### ORC Properties

| Property | Default | Purpose |
|----------|---------|---------|
| `compression-codec` | zlib | ORC compression |
| `stripe-size-bytes` | 64 MB | ORC stripe size (similar to Parquet row group) |
| `block-size-bytes` | 256 MB | HDFS block size |

### When to Use ORC

✅ **Use ORC when:**
- Legacy Hive integration required
- Existing ORC-based pipelines
- ACID transactions in Hive context
- Compatibility with Presto/Hive

❌ **Prefer Parquet when:**
- Multi-engine support (Spark, Athena, Snowflake)
- Better ecosystem support
- More active development

---

## Avro Configuration

### Basic Avro Setup

```sql
CREATE TABLE orders_avro (
    order_id BIGINT,
    customer_id BIGINT,
    order_data STRING
)
USING iceberg
TBLPROPERTIES (
    'write.format.default' = 'avro',
    'write.avro.compression-codec' = 'snappy'
);
```

### Avro Properties

| Property | Options | Default |
|----------|---------|---------|
| `compression-codec` | null, snappy, deflate, bzip2, xz, zstandard | snappy |

### When to Use Avro

✅ **Use Avro when:**
- Frequent schema evolution (Avro has best schema evolution)
- Row-oriented access patterns (full-row reads)
- Kafka integration (Avro is common Kafka format)
- Schema Registry integration

❌ **Prefer Parquet when:**
- Columnar queries (Parquet much faster)
- Large analytical scans
- Better compression needed

---

## File Format Decision Guide

| Scenario | Recommended Format | Compression | Why |
|----------|-------------------|-------------|-----|
| **General analytics** | Parquet | zstd | Best all-around performance |
| **Low-latency queries** | Parquet | snappy | Fast decompression |
| **High write throughput** | Parquet | lz4 | Minimal compression overhead |
| **Schema evolution heavy** | Avro | snappy | Native schema evolution |
| **Hive compatibility** | ORC | zstd | Legacy Hive integration |
| **Kafka → Iceberg CDC** | Avro → Parquet | snappy → zstd | Avro for schema flexibility, Parquet for queries |
| **Archival/cold storage** | Parquet | gzip | Maximum compression |
| **Interactive dashboards** | Parquet | snappy | Low query latency |

---

## Mixed Format Tables

### Use Different Formats by Partition

```python
# Write recent data as Avro (frequent schema changes)
recent_df.writeTo("orders") \
    .partitionedBy("order_date") \
    .option("write.format.default", "avro") \
    .append()

# Compact old data to Parquet (better query performance)
spark.sql("""
    CALL system.rewrite_data_files(
        table => 'orders',
        where => "order_date < current_date() - INTERVAL 30 DAYS",
        options => map('target-file-size-bytes', '536870912')
    )
""")

# Change format for old partitions
spark.sql("""
    ALTER TABLE orders SET TBLPROPERTIES (
        'write.format.default' = 'parquet',
        'write.parquet.compression-codec' = 'zstd'
    )
""")
```

---

## File Format Performance Comparison

### Query Performance (Columnar Scans)

| Format | SELECT COUNT(*) | SELECT specific columns | Full table scan |
|--------|----------------|------------------------|-----------------|
| Parquet (zstd) | 1.0x | 1.0x | 1.0x (baseline) |
| ORC (zstd) | 1.1x | 1.05x | 1.05x (5% slower) |
| Avro (snappy) | 8.0x | 6.0x | 5.0x (500% slower) |

**Context**: 1 billion row table, 20 columns, S3 storage

### Storage Efficiency

| Format | Compression Ratio (zstd) | Storage Cost |
|--------|------------------------|--------------|
| Parquet | 3.5:1 | $1.00 (baseline) |
| ORC | 3.2:1 | $1.09 (9% more) |
| Avro | 2.0:1 | $1.75 (75% more) |

### Write Performance

| Format | Write Throughput | CPU Usage |
|--------|-----------------|-----------|
| Parquet (lz4) | 500 MB/s | Medium |
| Parquet (zstd) | 300 MB/s | Medium-High |
| ORC (zstd) | 280 MB/s | Medium-High |
| Avro (snappy) | 600 MB/s | Low |

---

## Format Migration

### Migrate ORC → Parquet

```sql
-- Rewrite table from ORC to Parquet
CALL system.rewrite_data_files(
    table => 'orders_orc',
    options => map(
        'target-file-size-bytes', '536870912',
        'write.format.default', 'parquet',
        'write.parquet.compression-codec', 'zstd'
    )
);

-- Update table properties
ALTER TABLE orders_orc SET TBLPROPERTIES (
    'write.format.default' = 'parquet',
    'write.parquet.compression-codec' = 'zstd'
);
```

### Migrate Avro → Parquet (for old data)

```python
# Keep recent data as Avro (schema evolution)
# Convert old data to Parquet (query performance)

spark.sql("""
    CALL system.rewrite_data_files(
        table => 'events',
        where => "event_date < current_date() - INTERVAL 90 DAYS",
        options => map(
            'write.format.default', 'parquet',
            'write.parquet.compression-codec', 'zstd',
            'target-file-size-bytes', '536870912'
        )
    )
""")
```

---

## Troubleshooting File Format Issues

### Issue: Slow Query Performance

**Symptom**: Queries slower than expected

**Diagnosis**:
```sql
-- Check file format distribution
SELECT file_format, COUNT(*) AS file_count
FROM my_catalog.bronze_layer.orders.files
GROUP BY file_format;
```

**Solution**: Rewrite Avro files to Parquet

---

### Issue: High Storage Costs

**Symptom**: Storage costs higher than expected

**Diagnosis**:
```sql
-- Check compression codec
SELECT
    file_format,
    AVG(file_size_in_bytes) / 1024 / 1024 AS avg_file_size_mb,
    COUNT(*) AS file_count
FROM my_catalog.bronze_layer.orders.files
GROUP BY file_format;
```

**Solution**: Switch to higher compression (zstd or gzip)

---

### Issue: Slow Writes

**Symptom**: Write throughput lower than expected

**Solution**: Use lz4 or snappy compression instead of zstd/gzip

```sql
ALTER TABLE orders SET TBLPROPERTIES (
    'write.parquet.compression-codec' = 'lz4'
);
```

---

## References

- [Parquet Documentation](https://parquet.apache.org/docs/)
- [ORC Specification](https://orc.apache.org/specification/)
- [Avro Specification](https://avro.apache.org/docs/current/)
- [Iceberg File Formats](https://iceberg.apache.org/docs/latest/configuration/)
- [Compression Benchmark](https://github.com/facebook/zstd#benchmarks)
