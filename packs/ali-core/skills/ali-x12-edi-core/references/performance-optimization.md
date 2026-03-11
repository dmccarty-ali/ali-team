# ali-x12-edi-core - Performance Optimization Reference

## Performance Optimization

### Bulk Insert Pattern

```sql
-- Batch insert to master table
INSERT INTO NM1_MASTER
SELECT * FROM temp_parsed_nm1_batch;

-- Create split tables in single operation
CREATE TABLE nm1_2100_patient AS
SELECT * FROM NM1_MASTER
WHERE section_code ILIKE '%nm1_2100%';
```

### Indexing Strategy

**Master Tables:**
- Index on: section_code, parent_key_hash, file_key
- Supports split table creation and parent-child joins

**Split Tables:**
- Index on: key_hash, parent_key_hash, entity-specific IDs
- Optimized for analytical queries
