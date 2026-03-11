# ali-snowflake-data-engineer - Snowpark API Reference

## Snowpark DataFrame API

```python
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col, sum, avg, when, lit
from snowflake.snowpark.types import *

# Create session
session = Session.builder.configs(connection_params).create()

# Read table as DataFrame
df = session.table("sales")

# Transformations (lazy evaluation)
result = df \
    .filter(col("sale_date") >= "2024-01-01") \
    .group_by("region", "product_category") \
    .agg(
        sum("amount").alias("total_sales"),
        avg("amount").alias("avg_sale"),
        count("*").alias("num_transactions")
    ) \
    .filter(col("total_sales") > 10000) \
    .sort(col("total_sales").desc())

# Write results
result.write.mode("overwrite").save_as_table("sales_summary")

# Complex transformations with window functions
from snowflake.snowpark.functions import row_number
from snowflake.snowpark import Window

window = Window.partition_by("region").order_by(col("amount").desc())
ranked = df.with_column("rank", row_number().over(window))
top_sales = ranked.filter(col("rank") <= 10)
```

## Snowpark Stored Procedure

```python
# Define as stored procedure
from snowflake.snowpark import Session
from snowflake.snowpark.functions import *

def process_daily_data(session: Session, source_table: str, target_table: str) -> str:
    # Read source
    df = session.table(source_table)

    # Transform
    processed = df \
        .filter(col("process_date") == current_date()) \
        .with_column("processed_at", current_timestamp()) \
        .with_column("batch_id", lit(uuid4().hex))

    # Merge into target
    target = session.table(target_table)
    target.merge(
        processed,
        target["id"] == processed["id"],
        [
            when_matched().update({"amount": processed["amount"]}),
            when_not_matched().insert_all()
        ]
    )

    return f"Processed {processed.count()} rows"
```
