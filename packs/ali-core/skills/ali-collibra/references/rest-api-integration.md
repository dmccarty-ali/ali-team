# ali-collibra - REST API Integration Reference

## REST API Integration

### Comprehensive CRUD Example

```python
# Complete CollibraClient implementation
import requests
from typing import Dict, List

class CollibraClient:
    def __init__(self, base_url: str, api_token: str):
        self.base_url = base_url.rstrip("/")
        self.headers = {
            "Authorization": f"Bearer {api_token}",
            "Content-Type": "application/json"
        }

    def create_asset(
        self,
        domain_id: str,
        name: str,
        asset_type_id: str,
        attributes: Dict[str, str] = None
    ) -> str:
        """Create asset in Collibra."""
        payload = {
            "domainId": domain_id,
            "name": name,
            "typeId": asset_type_id
        }
        response = requests.post(
            f"{self.base_url}/rest/2.0/assets",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()
        asset_id = response.json()["id"]

        # Add attributes
        if attributes:
            for attr_type_id, value in attributes.items():
                self.add_attribute(asset_id, attr_type_id, value)

        return asset_id

    def add_attribute(
        self,
        asset_id: str,
        attribute_type_id: str,
        value: str
    ):
        """Add attribute to asset."""
        payload = {
            "assetId": asset_id,
            "typeId": attribute_type_id,
            "value": value
        }
        response = requests.post(
            f"{self.base_url}/rest/2.0/attributes",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()

    def create_relation(
        self,
        source_asset_id: str,
        target_asset_id: str,
        relation_type_id: str
    ):
        """Create relation between two assets."""
        payload = {
            "sourceId": source_asset_id,
            "targetId": target_asset_id,
            "typeId": relation_type_id
        }
        response = requests.post(
            f"{self.base_url}/rest/2.0/relations",
            json=payload,
            headers=self.headers
        )
        response.raise_for_status()

    def search_assets(
        self,
        query: str,
        asset_type_id: str = None,
        limit: int = 100
    ) -> List[Dict]:
        """Search for assets."""
        params = {
            "query": query,
            "limit": limit
        }
        if asset_type_id:
            params["typeId"] = asset_type_id

        response = requests.get(
            f"{self.base_url}/rest/2.0/assets",
            params=params,
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()["results"]
```

### REST API Endpoints Quick Reference

```
# Assets
GET    /rest/2.0/assets/{id}
POST   /rest/2.0/assets
PATCH  /rest/2.0/assets/{id}
DELETE /rest/2.0/assets/{id}

# Attributes
POST   /rest/2.0/attributes
PATCH  /rest/2.0/attributes/{id}

# Relations
POST   /rest/2.0/relations
DELETE /rest/2.0/relations/{id}

# Domains
GET    /rest/2.0/domains
POST   /rest/2.0/domains

# Communities
GET    /rest/2.0/communities
POST   /rest/2.0/communities

# Search
GET    /rest/2.0/search
```
