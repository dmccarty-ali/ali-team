# ali-snowflake-data-engineer - Advanced UDFs Reference

## Vectorized Python UDF (High Performance)

```sql
CREATE FUNCTION vectorized_sentiment(texts ARRAY)
RETURNS ARRAY
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
PACKAGES = ('pandas', 'textblob')
HANDLER = 'batch_analyze'
AS $$
import pandas as pd
from textblob import TextBlob

def batch_analyze(texts):
    return [TextBlob(t).sentiment.polarity if t else None for t in texts]
$$;
```

## UDTF (Table Function)

```sql
CREATE FUNCTION parse_log_entries(log_text VARCHAR)
RETURNS TABLE (
    timestamp TIMESTAMP,
    level VARCHAR,
    message VARCHAR
)
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
HANDLER = 'LogParser'
AS $$
import re
from datetime import datetime

class LogParser:
    def process(self, log_text):
        pattern = r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) \[(\w+)\] (.+)'
        for match in re.finditer(pattern, log_text):
            yield (
                datetime.strptime(match.group(1), '%Y-%m-%d %H:%M:%S'),
                match.group(2),
                match.group(3)
            )
$$;

-- Use UDTF
SELECT t.*
FROM logs,
     TABLE(parse_log_entries(log_content)) t;
```

## External Function (API Call)

```sql
-- Create API integration
CREATE API INTEGRATION geocoding_api
  API_PROVIDER = aws_api_gateway
  API_AWS_ROLE_ARN = 'arn:aws:iam::123456789:role/api-role'
  API_ALLOWED_PREFIXES = ('https://api.example.com/')
  ENABLED = TRUE;

-- Create external function
CREATE EXTERNAL FUNCTION geocode_address(address VARCHAR)
RETURNS VARIANT
API_INTEGRATION = geocoding_api
AS 'https://api.example.com/geocode';

-- Use in query
SELECT address, geocode_address(address) as geo_data
FROM locations;
```
