# ali-collibra - Custom Asset Types Reference

## Custom Asset Types

### Asset Type Definition

```yaml
# Custom Asset Type Example: API Endpoint
name: API Endpoint
description: REST API endpoint
parent_type: Technical Asset
attributes:
  - name: HTTP Method
    type: Single-value picklist
    values: [GET, POST, PUT, DELETE, PATCH]
  - name: URL Path
    type: Text
  - name: Authentication
    type: Single-value picklist
    values: [OAuth 2.0, API Key, None]
  - name: Rate Limit
    type: Numeric
  - name: Response Format
    type: Single-value picklist
    values: [JSON, XML, CSV]
relations:
  - name: produces
    target_type: Data Entity
  - name: consumes
    target_type: Data Entity
  - name: owned by
    target_type: Business Steward
```

### Creating via REST API

```python
import requests

COLLIBRA_URL = "https://your-instance.collibra.com/rest/2.0"
API_TOKEN = "your-api-token"

headers = {
    "Authorization": f"Bearer {API_TOKEN}",
    "Content-Type": "application/json"
}

# Create custom asset type
asset_type = {
    "name": "API Endpoint",
    "description": "REST API endpoint",
    "parentId": "<parent-asset-type-id>"
}
response = requests.post(
    f"{COLLIBRA_URL}/assetTypes",
    json=asset_type,
    headers=headers
)
asset_type_id = response.json()["id"]

# Add attributes
attribute = {
    "name": "HTTP Method",
    "kind": "SingleValueList",
    "allowedValues": ["GET", "POST", "PUT", "DELETE"]
}
requests.post(
    f"{COLLIBRA_URL}/attributeTypes",
    json=attribute,
    headers=headers
)
```

### Core Modules Overview

| Module | Purpose | Key Capabilities |
|--------|---------|------------------|
| **Data Catalog** | Technical metadata inventory | Tables, columns, schemas, databases |
| **Business Glossary** | Business terminology | Terms, definitions, stewardship assignments |
| **Data Lineage** | End-to-end data flow | Automated scanners for ETL/SQL, manual lineage |
| **Data Quality & Observability** | Quality measurement | Rule-based checks, scorecards, alerting |
| **Data Privacy** | PII/compliance tracking | GDPR/CCPA data subject tracking, retention policies |
| **Reference Data** | Master data management | Code lists, lookup tables, value standardization |
| **Data Marketplace** | Data discovery & access | Self-service catalog with request workflows |
