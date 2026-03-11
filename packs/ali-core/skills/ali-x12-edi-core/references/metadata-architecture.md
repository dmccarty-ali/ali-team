# ali-x12-edi-core - Metadata Architecture Reference

## Metadata-Driven Architecture

### tbl_defs.yaml Structure

Defines table structures, data types, and parsing rules:

```yaml
- table_name: NM1_MASTER
  sub_schema: lake_mededi
  columns:
    - column_name: NM101_ENTITY_ID_CODE
      data_type: VARCHAR
      max_length: 3
      required: true
    - column_name: NM109_ID_CODE
      data_type: VARCHAR
      max_length: 80
      required: false
```

### input_map.yaml Structure

Defines split table creation rules:

```yaml
- entry_num: 64
  table_name: nm1_2100_patient
  split_from_table: NM1_MASTER
  where_clause: "section_code ilike '%nm1_2100%'"
  sub_schema: stg_mededi
```

**Benefit:**
- No hardcoded table definitions
- Easy to add new split tables
- Self-documenting architecture
