# ali-collibra - Environment Management Reference

## Environment Management

### Environment Strategy

| Environment | Purpose | Refresh Frequency |
|-------------|---------|-------------------|
| **Development** | Configuration changes, testing | N/A (independent) |
| **Test** | UAT, workflow validation | Weekly from Dev |
| **Production** | Live governance platform | Promoted from Test |

### Configuration Promotion

```python
# Export/import workflow for config promotion
def promote_workflow(
    source_env: str,
    target_env: str,
    workflow_id: str
):
    """Promote workflow configuration between environments."""
    # Export from source
    source_client = CollibraClient(source_env, api_token)
    workflow_json = source_client.export_workflow(workflow_id)

    # Transform IDs (dev domain IDs → prod domain IDs)
    transformed = transform_ids(workflow_json, id_mapping)

    # Import to target
    target_client = CollibraClient(target_env, api_token)
    target_client.import_workflow(transformed)

def transform_ids(config: dict, mapping: dict) -> dict:
    """Transform environment-specific IDs."""
    for key, value in config.items():
        if key.endswith("Id") and value in mapping:
            config[key] = mapping[value]
    return config
```
