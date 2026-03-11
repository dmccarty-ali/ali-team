# ali-collibra - Lineage Scanners Reference

## Automated Lineage

### Out-of-the-Box Lineage Scanners

Collibra supports 30+ automated lineage scanners for:

**SQL Dialects:**
- Snowflake SQL
- Oracle PL/SQL
- SQL Server T-SQL
- PostgreSQL
- MySQL
- SAP HANA SQL

**ETL Tools:**
- Informatica PowerCenter
- Azure Data Factory
- AWS Glue
- Matillion
- Talend
- SSIS (SQL Server Integration Services)

**BI Tools:**
- Tableau
- Power BI
- Qlik
- Looker

### Lineage Capture Pattern

```
[Source Query History] → [Edge Lineage Scanner] → [Collibra API]
                              ↓
              Parse SQL AST, extract:
                - SELECT columns → SOURCE
                - FROM tables → ORIGIN
                - JOIN conditions → RELATIONS
                - WHERE filters → TRANSFORMATIONS
                - INSERT INTO → TARGET
```

**Example:**
```sql
-- This query in Snowflake query history:
INSERT INTO EDW_MEDEDI.FACT_CLAIM
SELECT
    c.claim_id,
    p.provider_npi,
    SUM(amt.charge_amount) AS total_charge
FROM STG_MEDEDI.CLAIM c
JOIN STG_MEDEDI.PROVIDER p ON c.provider_id = p.provider_id
JOIN STG_MEDEDI.CHARGE amt ON c.claim_id = amt.claim_id
GROUP BY c.claim_id, p.provider_npi;

-- Lineage scanner creates:
FACT_CLAIM.claim_id ← CLAIM.claim_id
FACT_CLAIM.provider_npi ← PROVIDER.provider_npi
FACT_CLAIM.total_charge ← CHARGE.charge_amount (aggregated)
```

### Automated vs Manual Lineage Decision Guide

| Scenario | Automated Lineage | Manual Lineage |
|----------|------------------|----------------|
| SQL-based transformations | ✓ | |
| Informatica ETL | ✓ | |
| Custom Python scripts | | ✓ |
| Spreadsheet transforms | | ✓ |
| Legacy mainframe | | ✓ |
