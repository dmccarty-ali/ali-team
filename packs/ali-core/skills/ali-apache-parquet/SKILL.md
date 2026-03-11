---
name: ali-apache-parquet
description: |
  Apache Parquet file format patterns for data engineering. Use when:

  PLANNING: Designing Parquet schemas, planning file partitioning, evaluating
  compression options, sizing files for Snowflake/Spark ingestion

  IMPLEMENTATION: Writing Parquet files with PyArrow or Java, reading Parquet,
  converting from other formats, implementing schema evolution

  GUIDANCE: Asking about Parquet best practices, optimal file sizes, compression
  choices, nested types, row group sizing, Snowflake COPY patterns

  REVIEW: Reviewing Parquet schema design, checking file sizes, validating
  compression settings, auditing partitioning strategy
---

# Apache Parquet

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Parquet schema for a data pipeline
- Planning file partitioning strategy
- Choosing compression algorithm
- Sizing files for target system (Snowflake, Spark)

**Implementation:**
- Writing Parquet files with PyArrow
- Writing Parquet files with Java
- Reading and processing Parquet files
- Schema evolution (adding/removing columns)

**Guidance/Best Practices:**
- Asking about optimal file sizes
- Asking about compression options
- Asking about nested types and arrays
- Asking about Snowflake COPY integration

**Review/Validation:**
- Reviewing Parquet schema definitions
- Checking row group configuration
- Validating compression choices
- Auditing file organization

---

## Key Principles

- **Columnar storage**: Parquet excels at reading subset of columns
- **Right-size files**: 100MB-1GB optimal for most systems
- **Match schema to queries**: Flatten if always reading all fields
- **Compression matters**: Snappy for speed, ZSTD for size
- **Row groups for parallelism**: ~100MB row groups enable parallel reads
- **Schema evolution**: Add columns freely; removing requires care

---

## Schema Design

### Basic Types

```python
import pyarrow as pa

# Primitive types
schema = pa.schema([
    ('claim_id', pa.string()),
    ('patient_id', pa.string()),
    ('service_date', pa.date32()),
    ('charge_amount', pa.decimal128(12, 2)),
    ('units', pa.int32()),
    ('is_paid', pa.bool_()),
    ('created_at', pa.timestamp('us', tz='UTC')),
])
```

### Nested Types

```python
# Struct (nested object)
address_type = pa.struct([
    ('street', pa.string()),
    ('city', pa.string()),
    ('state', pa.string()),
    ('zip', pa.string()),
])

# List (array)
diagnosis_list = pa.list_(pa.string())

# Map (key-value pairs)
metadata_map = pa.map_(pa.string(), pa.string())

schema = pa.schema([
    ('claim_id', pa.string()),
    ('patient_address', address_type),
    ('diagnosis_codes', diagnosis_list),
    ('metadata', metadata_map),
])
```

### Nullable vs Required

```python
# Nullable (default) - can contain nulls
pa.field('optional_field', pa.string(), nullable=True)

# Required - cannot contain nulls
pa.field('required_field', pa.string(), nullable=False)
```

---

## Compression Options

| Algorithm | Speed | Ratio | Use Case |
|-----------|-------|-------|----------|
| SNAPPY | Fastest | Lower | Default, balanced |
| ZSTD | Fast | Higher | Storage optimization |
| GZIP | Slower | High | Compatibility, archival |
| LZ4 | Very Fast | Lower | Speed-critical |
| None | N/A | N/A | Testing only |

```python
import pyarrow.parquet as pq

# Write with ZSTD compression (best balance)
pq.write_table(
    table,
    'output.parquet',
    compression='ZSTD',
    compression_level=3,  # 1-22, higher = smaller but slower
)

# Write with Snappy (fastest)
pq.write_table(
    table,
    'output.parquet',
    compression='SNAPPY',
)
```

---

## Row Group Sizing

```python
# Row group size affects:
# - Memory usage during write
# - Parallelism during read
# - Predicate pushdown effectiveness

# Target: ~100MB row groups
pq.write_table(
    table,
    'output.parquet',
    row_group_size=100_000,  # Rows per group (adjust based on row size)
)

# For Snowflake: 100-250MB row groups optimal
# For Spark: Match to partition size
```

---

## PyArrow Patterns

### Writing Parquet

```python
import pyarrow as pa
import pyarrow.parquet as pq
from datetime import date
from decimal import Decimal

# From Python data
data = {
    'claim_id': ['CLM001', 'CLM002', 'CLM003'],
    'patient_id': ['PAT001', 'PAT002', 'PAT001'],
    'service_date': [date(2024, 1, 15), date(2024, 1, 16), date(2024, 1, 17)],
    'charge_amount': [Decimal('150.00'), Decimal('200.50'), Decimal('75.00')],
}

table = pa.Table.from_pydict(data, schema=schema)

# Write single file
pq.write_table(table, 'claims.parquet', compression='ZSTD')

# Write partitioned dataset
pq.write_to_dataset(
    table,
    root_path='claims_partitioned/',
    partition_cols=['service_date'],
    compression='ZSTD',
)
```

### Reading Parquet

```python
import pyarrow.parquet as pq

# Read entire file
table = pq.read_table('claims.parquet')
df = table.to_pandas()

# Read specific columns only
table = pq.read_table('claims.parquet', columns=['claim_id', 'charge_amount'])

# Read with filter (predicate pushdown)
table = pq.read_table(
    'claims.parquet',
    filters=[('service_date', '>=', date(2024, 1, 1))],
)

# Read partitioned dataset
dataset = pq.ParquetDataset('claims_partitioned/')
table = dataset.read()

# Stream large files
parquet_file = pq.ParquetFile('large_file.parquet')
for batch in parquet_file.iter_batches(batch_size=10000):
    process_batch(batch.to_pandas())
```

### Inspect Parquet Metadata

```python
import pyarrow.parquet as pq

# File metadata
parquet_file = pq.ParquetFile('claims.parquet')
print(f"Schema: {parquet_file.schema_arrow}")
print(f"Row groups: {parquet_file.metadata.num_row_groups}")
print(f"Total rows: {parquet_file.metadata.num_rows}")

# Row group details
for i in range(parquet_file.metadata.num_row_groups):
    rg = parquet_file.metadata.row_group(i)
    print(f"Row group {i}: {rg.num_rows} rows, {rg.total_byte_size} bytes")
```

---

## Java Parquet Patterns

### Writing with ParquetWriter

```java
import org.apache.parquet.hadoop.ParquetWriter;
import org.apache.parquet.hadoop.metadata.CompressionCodecName;
import org.apache.parquet.avro.AvroParquetWriter;
import org.apache.hadoop.fs.Path;

// Using Avro schema
Schema schema = new Schema.Parser().parse(new File("claim.avsc"));

try (ParquetWriter<GenericRecord> writer = AvroParquetWriter
        .<GenericRecord>builder(new Path("claims.parquet"))
        .withSchema(schema)
        .withCompressionCodec(CompressionCodecName.ZSTD)
        .withRowGroupSize(100 * 1024 * 1024)  // 100MB row groups
        .build()) {

    for (Claim claim : claims) {
        GenericRecord record = new GenericData.Record(schema);
        record.put("claim_id", claim.getId());
        record.put("charge_amount", claim.getAmount());
        writer.write(record);
    }
}
```

### Reading with ParquetReader

```java
import org.apache.parquet.hadoop.ParquetReader;
import org.apache.parquet.avro.AvroParquetReader;

try (ParquetReader<GenericRecord> reader = AvroParquetReader
        .<GenericRecord>builder(new Path("claims.parquet"))
        .build()) {

    GenericRecord record;
    while ((record = reader.read()) != null) {
        String claimId = record.get("claim_id").toString();
        // Process record
    }
}
```

### Snowpipe Streaming with Parquet

```java
// For ali-ai-ingestion-engine parser pattern
import net.snowflake.ingest.streaming.*;

// Build Parquet in memory, stream to Snowflake
ByteArrayOutputStream baos = new ByteArrayOutputStream();
try (ParquetWriter<GenericRecord> writer = AvroParquetWriter
        .<GenericRecord>builder(new OutputFile() {
            // Custom OutputFile implementation for in-memory
        })
        .withSchema(schema)
        .withCompressionCodec(CompressionCodecName.SNAPPY)
        .build()) {

    for (ParsedClaim claim : claims) {
        writer.write(toRecord(claim));
    }
}

// Upload to stage, COPY INTO table
byte[] parquetBytes = baos.toByteArray();
```

---

## Partitioned Datasets

### Directory Structure

```
# Hive-style partitioning
claims/
├── service_date=2024-01-01/
│   ├── part-00000.parquet
│   └── part-00001.parquet
├── service_date=2024-01-02/
│   └── part-00000.parquet
└── service_date=2024-01-03/
    └── part-00000.parquet

# Custom partitioning
claims/
├── 2024/
│   ├── 01/
│   │   ├── 01/
│   │   │   └── claims.parquet
│   │   └── 02/
│   │       └── claims.parquet
```

### Writing Partitioned Data

```python
import pyarrow.parquet as pq

# Single partition column
pq.write_to_dataset(
    table,
    root_path='claims/',
    partition_cols=['service_date'],
    existing_data_behavior='delete_matching',  # Overwrite matching partitions
)

# Multiple partition columns
pq.write_to_dataset(
    table,
    root_path='claims/',
    partition_cols=['year', 'month', 'day'],
    compression='ZSTD',
)
```

### Reading Partitioned Data

```python
import pyarrow.dataset as ds

# Read entire dataset
dataset = ds.dataset('claims/', format='parquet', partitioning='hive')
table = dataset.to_table()

# Read with partition filter (very efficient - skips files)
table = dataset.to_table(
    filter=(ds.field('service_date') >= '2024-01-01')
)
```

---

## Schema Evolution

### Adding Columns (Safe)

```python
# Original schema
schema_v1 = pa.schema([
    ('claim_id', pa.string()),
    ('amount', pa.decimal128(12, 2)),
])

# Add new column with default
schema_v2 = pa.schema([
    ('claim_id', pa.string()),
    ('amount', pa.decimal128(12, 2)),
    ('status', pa.string()),  # New column - nullable
])

# Reading v1 files with v2 schema returns null for new column
# This is safe and automatic
```

### Removing Columns (Careful)

```python
# Removing columns from schema is fine
# Old files still contain the column data (wasted space)
# Readers just ignore columns not in schema

# To truly remove: rewrite files with new schema
```

### Type Changes (Avoid)

```python
# Type changes are NOT automatically compatible
# Must rewrite all data
# Exception: widening numeric types (int32 -> int64)
```

---

## Snowflake Integration

### Optimal File Sizing

```python
# Snowflake COPY works best with:
# - Files: 100-250MB compressed
# - Row groups: 100MB
# - Compression: AUTO (Snappy) or ZSTD

# Write optimized for Snowflake
pq.write_table(
    table,
    'for_snowflake.parquet',
    compression='SNAPPY',  # Snowflake default
    row_group_size=500_000,  # Adjust based on row width
)
```

### COPY INTO Pattern

```sql
-- Stage files first
PUT file://local_path/*.parquet @my_stage/path/

-- COPY with Parquet format
COPY INTO my_table
FROM @my_stage/path/
FILE_FORMAT = (TYPE = PARQUET)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE
ON_ERROR = CONTINUE;

-- With transformation
COPY INTO my_table (col1, col2, col3)
FROM (
    SELECT
        $1:claim_id::STRING,
        $1:amount::NUMBER(12,2),
        $1:service_date::DATE
    FROM @my_stage/path/
)
FILE_FORMAT = (TYPE = PARQUET);
```

### Schema Matching

```python
# Column names in Parquet should match Snowflake table
# Use MATCH_BY_COLUMN_NAME for flexibility

# Snowflake reads nested Parquet as VARIANT
# Flatten in COPY or query:
# SELECT data:nested_field::STRING FROM table
```

---

## File Size Guidelines

| System | Optimal File Size | Row Groups |
|--------|-------------------|------------|
| Snowflake | 100-250MB | 100MB |
| Spark | 128MB-1GB | 128MB (match block size) |
| Athena/Presto | 100-500MB | 100MB |
| S3 Select | <128MB | Smaller |

### Calculating File Size

```python
import pyarrow.parquet as pq

def estimate_file_size(table, compression='ZSTD'):
    """Estimate compressed Parquet size."""
    # Write to memory buffer
    import io
    buffer = io.BytesIO()
    pq.write_table(table, buffer, compression=compression)
    return buffer.tell()

# Target rows per file for ~100MB files
sample_table = table.slice(0, 1000)
sample_size = estimate_file_size(sample_table)
rows_per_file = int(100_000_000 / (sample_size / 1000))
```

---

## Performance Tips

### Column Pruning

```python
# Only read needed columns - huge performance gain
# BAD: Read all columns
table = pq.read_table('large_file.parquet')
df = table.to_pandas()[['col1', 'col2']]

# GOOD: Read only needed columns
table = pq.read_table('large_file.parquet', columns=['col1', 'col2'])
df = table.to_pandas()
```

### Predicate Pushdown

```python
# Filters applied at read time, skipping row groups
# Works best with sorted data

# Sort data by commonly filtered column before writing
table = table.sort_by('service_date')
pq.write_table(table, 'sorted.parquet')

# Then filter efficiently
table = pq.read_table(
    'sorted.parquet',
    filters=[('service_date', '>=', date(2024, 1, 1))]
)
```

### Memory Management

```python
# For large files, use batched reading
parquet_file = pq.ParquetFile('huge_file.parquet')

# Process in chunks
for batch in parquet_file.iter_batches(batch_size=100_000):
    df = batch.to_pandas()
    process(df)
    del df  # Free memory
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Tiny files (<10MB) | Overhead per file | Combine into larger files |
| Huge files (>1GB) | Can't parallelize | Split into 100-250MB |
| No compression | Wasted storage/bandwidth | Use SNAPPY or ZSTD |
| Row group = entire file | No parallelism | 100MB row groups |
| Deep nesting | Query complexity | Flatten for common queries |
| Frequent schema changes | Compatibility issues | Plan schema upfront |
| SELECT * on wide tables | Reads all columns | Select needed columns |

---

## Avro vs Parquet

| Aspect | Avro | Parquet |
|--------|------|---------|
| **Format** | Row-based | Column-based |
| **Best for** | Write-heavy, streaming | Read-heavy, analytics |
| **Compression** | Good (block level) | Excellent (column level) |
| **Schema** | Embedded in file | Embedded in file |
| **Splittable** | Yes | Yes |
| **Partial reads** | Must read full row | Can read single columns |
| **Use case** | Kafka, data pipelines | Data warehouse, BI |

```python
# When to use Avro
# - Streaming/Kafka (row-at-a-time)
# - Schema evolution is frequent
# - Write performance critical
# - Need to read entire records

# When to use Parquet
# - Analytics queries (SELECT specific columns)
# - Storage efficiency matters
# - Query engines (Snowflake, Spark, Athena)
# - Aggregations on subset of columns
```

---

## Delta Lake / Iceberg Integration

Table formats for ACID transactions on data lakes:

### Delta Lake (Databricks)

```python
# Write Delta table
df.write.format("delta").save("/data/claims")

# Read Delta table
df = spark.read.format("delta").load("/data/claims")

# Time travel
df = spark.read.format("delta").option("versionAsOf", 5).load("/data/claims")
df = spark.read.format("delta").option("timestampAsOf", "2024-01-01").load("/data/claims")

# MERGE (upsert)
deltaTable.merge(
    updates_df.alias("updates"),
    "claims.claim_id = updates.claim_id"
).whenMatchedUpdate(...).whenNotMatchedInsert(...).execute()
```

### Apache Iceberg (Snowflake native support)

```sql
-- Create Iceberg table in Snowflake
CREATE ICEBERG TABLE claims_iceberg (
    claim_id STRING,
    amount NUMBER(12,2),
    service_date DATE
)
CATALOG = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'my_s3_volume'
BASE_LOCATION = 'claims/';

-- Time travel (Iceberg snapshots)
SELECT * FROM claims_iceberg AT(TIMESTAMP => '2024-01-01 00:00:00');

-- Benefits over standard Parquet:
-- - ACID transactions
-- - Schema evolution
-- - Time travel / versioning
-- - Efficient upserts (not full rewrites)
-- - Hidden partitioning
```

---

## Decimal Precision for Financial Data

```python
import pyarrow as pa

# Financial amount fields - use decimal, not float!
# Avoid: pa.float64() - has precision issues

# Standard precision options
amount_12_2 = pa.decimal128(12, 2)   # Up to 9,999,999,999.99
amount_14_4 = pa.decimal128(14, 4)   # For rates, higher precision
amount_18_2 = pa.decimal128(18, 2)   # Very large amounts

schema = pa.schema([
    ('claim_id', pa.string()),
    ('charge_amount', pa.decimal128(12, 2)),
    ('paid_amount', pa.decimal128(12, 2)),
    ('adjustment_rate', pa.decimal128(8, 6)),  # e.g., 0.123456
])

# Always use Decimal in Python, not float
from decimal import Decimal

data = {
    'claim_id': ['CLM001'],
    'charge_amount': [Decimal('1234.56')],  # Not 1234.56
    'paid_amount': [Decimal('1000.00')],
}
```

---

## Timestamp and Timezone Handling

```python
import pyarrow as pa
from datetime import datetime, timezone

# Timestamp types
ts_ntz = pa.timestamp('us')           # No timezone (local/naive)
ts_utc = pa.timestamp('us', tz='UTC') # With timezone

# Snowflake compatibility
# TIMESTAMP_NTZ -> pa.timestamp('us') or pa.timestamp('ns')
# TIMESTAMP_TZ  -> pa.timestamp('us', tz='UTC')
# TIMESTAMP_LTZ -> pa.timestamp('us', tz='UTC') (stored as UTC)

schema = pa.schema([
    ('event_id', pa.string()),
    ('event_time', pa.timestamp('us', tz='UTC')),     # Best for events
    ('business_date', pa.date32()),                    # Date only
    ('created_at', pa.timestamp('us')),                # NTZ for system times
])

# Converting Python datetimes
from datetime import datetime, timezone

# Always use timezone-aware for timestamps
event_time = datetime.now(timezone.utc)

# For date-only fields
from datetime import date
business_date = date(2024, 1, 15)
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Java Parquet writing | `parsers/edi-common/src/.../ParquetWriter.java` |
| Python Parquet handling | `sql_gen_build/` (generated data) |
| Parquet schemas | `parsers/edi-*/schemas/` |
| Snowflake COPY SQL | `sql_gen_build/output/data_flow_*.sql` |

---

## References

- [Apache Parquet Documentation](https://parquet.apache.org/docs/)
- [PyArrow Documentation](https://arrow.apache.org/docs/python/)
- [Snowflake Parquet Loading](https://docs.snowflake.com/en/user-guide/data-load-parquet)
- [Parquet File Format](https://github.com/apache/parquet-format)
