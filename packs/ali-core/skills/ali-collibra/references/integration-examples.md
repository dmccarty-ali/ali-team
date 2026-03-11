# ali-collibra - Integration Examples Reference

## Specific Integration Patterns

### Collibra-to-SAP MDG Integration

```python
# Sync data governance policies from Collibra to SAP MDG
def sync_policies_to_sap_mdg(
    collibra_client: CollibraClient,
    sap_mdg_client,
    domain_id: str
):
    """
    Sync governance policies from Collibra to SAP MDG.

    Pattern:
    1. Query Collibra for data quality rules in domain
    2. Transform to SAP MDG policy format
    3. POST to SAP MDG REST API
    """
    # Get Collibra data quality rules
    rules = collibra_client.search_assets(
        query="",
        asset_type_id="<data-quality-rule-type-id>"
    )

    for rule in rules:
        # Extract rule definition
        rule_name = rule["name"]
        rule_logic = get_attribute(rule["id"], "Rule Logic")
        threshold = get_attribute(rule["id"], "Threshold")

        # Transform to SAP MDG format
        sap_policy = {
            "policyName": rule_name,
            "validationLogic": rule_logic,
            "threshold": threshold,
            "source": "Collibra"
        }

        # POST to SAP MDG
        sap_mdg_client.create_policy(sap_policy)
```

### Azure AD / SAML SSO Integration

```yaml
# Azure AD SAML Configuration (conceptual)
saml_config:
  identity_provider:
    entity_id: https://sts.windows.net/{tenant-id}/
    sso_url: https://login.microsoftonline.com/{tenant-id}/saml2
    certificate: |
      -----BEGIN CERTIFICATE-----
      [Azure AD signing certificate]
      -----END CERTIFICATE-----

  service_provider:
    entity_id: https://your-instance.collibra.com
    acs_url: https://your-instance.collibra.com/saml/SSO
    name_id_format: urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress

  attribute_mapping:
    email: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress
    first_name: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname
    last_name: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname
    groups: http://schemas.microsoft.com/ws/2008/06/identity/claims/groups
```

### Informatica Lineage Integration

```python
# Extract lineage from Informatica PowerCenter via API
def extract_informatica_lineage(
    informatica_client,
    collibra_client: CollibraClient
):
    """
    Extract lineage from Informatica and push to Collibra.

    Pattern:
    1. Query Informatica repository for mappings
    2. Extract source → transformation → target relationships
    3. Create lineage relations in Collibra
    """
    mappings = informatica_client.get_mappings()

    for mapping in mappings:
        # Get source tables
        sources = mapping.get_sources()
        # Get target tables
        targets = mapping.get_targets()
        # Get transformations
        transformations = mapping.get_transformations()

        # Create assets in Collibra
        for source in sources:
            source_asset_id = collibra_client.create_asset(
                domain_id="<lineage-domain>",
                name=source["table_name"],
                asset_type_id="<table-asset-type>"
            )

        for target in targets:
            target_asset_id = collibra_client.create_asset(
                domain_id="<lineage-domain>",
                name=target["table_name"],
                asset_type_id="<table-asset-type>"
            )

            # Create lineage relation
            collibra_client.create_relation(
                source_asset_id=source_asset_id,
                target_asset_id=target_asset_id,
                relation_type_id="<lineage-relation-type>"
            )
```

### Azure Data Factory Lineage Integration

```python
# Extract lineage from Azure Data Factory
def extract_adf_lineage(
    adf_client,
    collibra_client: CollibraClient
):
    """Extract lineage from ADF pipelines."""
    pipelines = adf_client.list_pipelines()

    for pipeline in pipelines:
        activities = pipeline.get_activities()

        for activity in activities:
            if activity.type == "Copy":
                source_dataset = activity.inputs[0]
                sink_dataset = activity.outputs[0]

                # Create lineage in Collibra
                source_id = find_or_create_asset(
                    collibra_client,
                    source_dataset.name
                )
                sink_id = find_or_create_asset(
                    collibra_client,
                    sink_dataset.name
                )

                collibra_client.create_relation(
                    source_asset_id=source_id,
                    target_asset_id=sink_id,
                    relation_type_id="<lineage-relation-type>"
                )
```
