# ali-x12-edi-core - Error Handling Reference

## Error Handling Best Practices

### File-Level Errors

**Invalid ISA:** Reject entire interchange
**Missing GS/GE:** Reject functional group
**Invalid ST:** Reject transaction set

### Segment-Level Errors

**Missing Required Segment:** Flag transaction, continue parsing
**Invalid Segment Structure:** Log error, skip segment
**Invalid Data Element:** Use NULL, log warning

### Data Quality Issues

**Unknown Entity Codes:** Flag for review, store raw value
**Missing Required Elements:** Use NULL, flag transaction
**Format Violations:** Attempt to parse, log warning
