# ali-x12-edi-core - Version Compatibility Reference

## X12 Version Compatibility

### HIPAA 5010 Standards

**Current Version:** 005010 (implemented 2012)
**Common Transactions:**
- 835: 005010X221A1 (Health Care Claim Payment/Advice)
- 837i: 005010X223A2 (Institutional Claims)
- 837p: 005010X222A2 (Professional Claims)

**Key Changes from 4010:**
- Support for ICD-10 diagnosis codes
- NPI (National Provider Identifier) required
- Enhanced coordination of benefits support
- Expanded address fields

### Version Identification

Check GS08 segment element for version:
```
GS*HP*SENDER*RECEIVER*20201231*1234*1*X*005010X221A1~
                                                ^^^^^^^^^^^^
                                                Version ID
```
