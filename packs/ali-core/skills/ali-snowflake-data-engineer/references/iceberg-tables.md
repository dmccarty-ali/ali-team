# ali-snowflake-data-engineer - Iceberg Tables Reference

## Native Iceberg Table Support

```sql
-- Create Iceberg table with Snowflake catalog
CREATE ICEBERG TABLE my_iceberg_table (
    id INT,
    name STRING,
    created_at TIMESTAMP
)
CATALOG = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'my_external_volume'
BASE_LOCATION = 'iceberg_data/';

-- Create from external Iceberg catalog
CREATE ICEBERG TABLE ext_iceberg
  CATALOG = 'my_glue_catalog'
  CATALOG_TABLE_NAME = 'source_table';

-- Schema evolution
ALTER ICEBERG TABLE my_iceberg_table ADD COLUMN new_col STRING;

-- Time travel (uses Iceberg snapshots)
SELECT * FROM my_iceberg_table AT (TIMESTAMP => '2024-01-15 10:00:00');
```

## External Volume Setup

```sql
CREATE EXTERNAL VOLUME my_external_volume
  STORAGE_LOCATIONS = (
    (
      NAME = 's3_location'
      STORAGE_PROVIDER = 'S3'
      STORAGE_BASE_URL = 's3://my-bucket/iceberg/'
      STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789:role/snowflake-role'
    )
  );
```
