# ali-wherescape - Lineage Tracking Reference

This reference provides detailed guidance on WhereScape's lineage tracking capabilities and Collibra integration.

---

## Built-In Lineage Capabilities

### Column-level lineage
- Source column → Target column mappings
- Transformation logic captured in metadata
- Lineage graph visualization in WhereScape UI

### Table-level lineage
- Source table → Target table dependencies
- Job dependencies tracked
- Impact analysis for schema changes

### Export formats
- CSV export of lineage metadata
- XML export for integration
- REST API for programmatic access

---

## Lineage Integration with Collibra

**Crown Equipment context:** REQ-F-100 and REQ-F-101 require WhereScape lineage integration with Collibra.

### Integration Approaches

| Approach | Method | Complexity |
|----------|--------|------------|
| **CSV Export** | Export WhereScape lineage to CSV, import into Collibra | Low |
| **REST API** | Query WhereScape metadata API, push to Collibra API | Medium |
| **Collibra Integration** | Use Collibra Edge for automated sync | High (requires Collibra Edge) |

---

## CSV Export Structure

```csv
source_schema,source_table,source_column,target_schema,target_table,target_column,transformation_logic
LAKE,CUSTOMER_RAW,cust_id,STG,CUSTOMER,customer_id,Direct mapping
LAKE,CUSTOMER_RAW,cust_name,STG,CUSTOMER,customer_name,UPPER(cust_name)
STG,CUSTOMER,customer_id,EDW,DIM_CUSTOMER,customer_id,Direct mapping
STG,CUSTOMER,customer_name,EDW,DIM_CUSTOMER,customer_name,Direct mapping
```

---

## Collibra API Push Example

```python
import requests
import json

# Query WhereScape lineage API
ws_lineage = requests.get('http://wherescape-server/api/lineage/table/STG_CUSTOMER')
lineage_data = ws_lineage.json()

# Transform to Collibra format
collibra_lineage = []
for mapping in lineage_data['column_mappings']:
    collibra_lineage.append({
        'sourceAssetId': mapping['source_column_id'],
        'targetAssetId': mapping['target_column_id'],
        'relationType': 'Column to Column Lineage',
        'transformationLogic': mapping['transformation']
    })

# Push to Collibra
collibra_response = requests.post(
    'https://collibra-instance/rest/2.0/relations',
    headers={'Authorization': 'Bearer {token}'},
    json=collibra_lineage
)
```

---

**Document Version:** 1.0
**Last Updated:** 2026-02-16
**Maintained By:** ALI AI Team
