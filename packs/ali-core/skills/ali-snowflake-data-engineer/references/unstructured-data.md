# ali-snowflake-data-engineer - Unstructured Data Reference

## Directory Tables

```sql
-- Create directory table for file metadata
CREATE STAGE doc_stage
  DIRECTORY = (ENABLE = TRUE);

-- List files with metadata
SELECT *
FROM DIRECTORY(@doc_stage);

-- Pre-signed URL for file access
SELECT
    relative_path,
    GET_PRESIGNED_URL(@doc_stage, relative_path, 3600) as download_url
FROM DIRECTORY(@doc_stage)
WHERE relative_path LIKE '%.pdf';
```

## PDF Processing

```sql
-- Process with Python UDF
CREATE FUNCTION extract_pdf_text(file_url VARCHAR)
RETURNS VARCHAR
LANGUAGE PYTHON
RUNTIME_VERSION = '3.10'
PACKAGES = ('pypdf2', 'requests')
HANDLER = 'extract'
AS $$
import requests
from PyPDF2 import PdfReader
from io import BytesIO

def extract(url):
    response = requests.get(url)
    reader = PdfReader(BytesIO(response.content))
    return ' '.join(page.extract_text() for page in reader.pages)
$$;
```
