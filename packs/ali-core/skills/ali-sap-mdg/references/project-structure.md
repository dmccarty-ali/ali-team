# ali-sap-mdg - Project Structure Reference

This reference contains template project structures and directory organization patterns for SAP MDG implementations.

---

## Project Structure Template

**Note:** SAP MDG is a commercial product; we do not maintain MDG source code. Configuration and extensions are project-specific.

---

## Recommended Directory Structure

```
/config/
  mdg/
    brf_rules/                 BRFplus decision table exports
      DT_SINGLE_VAL/           Process flow decision tables
      DT_USER_AGT_GRP/         Agent determination tables
      DT_CONDITIONS/           Conditional routing tables
    matching_rules/            Matching configuration (USMD_RULE exports)
      deterministic/           Exact match rules
      probabilistic/           Fuzzy match rules
      survivorship/            Survivorship logic
    data_model/                Custom entity definitions (USMD_MODEL exports)
      customer/                Customer domain extensions
      material/                Material domain extensions
      supplier/                Supplier domain extensions
    workflows/                 Workflow configuration documentation
      customer_change.md       Customer change workflow spec
      material_change.md       Material change workflow spec
    integrations/              MDI/IDoc configuration specs
      mdi/                     SAP Master Data Integration configs
      idoc/                    IDoc distribution models
      api/                     REST/OData API documentation

/docs/
  mdg/
    architecture.md            Hub vs co-deployment decisions
    domain_design.md           Domain-specific data model and rules
    transport_plan.md          Transport route and cutover plan
    matching_strategy.md       Matching and consolidation approach
    integration_strategy.md    Distribution patterns and target systems
    user_guide/                End-user documentation
      steward_guide.md         Data steward procedures
      approver_guide.md        Approval workflows
    admin_guide/               Administrative documentation
      transport_procedure.md   How to create/release transports
      monitoring.md            How to monitor MDG health

/scripts/
  mdg/
    data_migration/            Data migration scripts (legacy to MDG)
      extract_legacy.py        Extract from ECC/legacy systems
      transform_data.py        Cleanse and transform
      load_mdg.py              Load into MDG hub
    test_data/                 Test data generation for UAT
      generate_customers.py    Generate test customers
      generate_materials.py    Generate test materials
    monitoring/                Monitoring and alerting scripts
      check_workflow_queue.py  Alert on backed-up approval queues
      check_integration.py     Validate MDI/IDoc status

/tests/
  mdg/
    unit/                      Unit tests (if extending MDG with custom code)
    integration/               Integration tests
      test_workflow.py         Test BRFplus workflow execution
      test_matching.py         Test matching accuracy
      test_integration.py      Test MDI/IDoc distribution
    performance/               Performance tests
      test_matching_scale.py   Matching with 100K records
      test_concurrent_users.py Simulate 100 concurrent users

/transports/                   Transport request documentation
  DEVK900120.md                TR details: Add custom field CREDIT_RATING
  DEVK900121.md                TR details: Update BRFplus to use CREDIT_RATING
  DEVK900122.md                TR details: Update UI for CREDIT_RATING
  transport_dependencies.md    Document which TRs depend on each other
```

---

## Configuration Files

### BRFplus Rule Exports

**Location:** `/config/mdg/brf_rules/`

**Format:** XML exports from USMD_SSW_RULE

**Example directory:**
```
/config/mdg/brf_rules/
  customer_change/
    DT_SINGLE_VAL_CUSTOMER_CHANGE.xml
    DT_USER_AGT_GRP_CUSTOMER_CHANGE.xml
    DT_CONDITIONS_CUSTOMER_CHANGE.xml
  material_change/
    DT_SINGLE_VAL_MATERIAL_CHANGE.xml
    DT_USER_AGT_GRP_MATERIAL_CHANGE.xml
    DT_CONDITIONS_MATERIAL_CHANGE.xml
```

---

### Matching Rule Exports

**Location:** `/config/mdg/matching_rules/`

**Format:** YAML or JSON (documented, not SAP native export)

**Example:**
```yaml
# /config/mdg/matching_rules/deterministic/customer_tax_id.yaml
rule_name: MATCH_CUSTOMER_TAX_ID
rule_type: deterministic
entity: Customer
match_key: TAX_ID
threshold: 100  # Exact match required
active: true
```

---

### Data Model Extensions

**Location:** `/config/mdg/data_model/`

**Format:** XML exports from USMD_MODEL

**Example:**
```xml
<!-- /config/mdg/data_model/customer/CREDIT_RATING.xml -->
<entity_attribute>
  <entity>CUSTOMER</entity>
  <attribute>CREDIT_RATING</attribute>
  <data_type>CHAR(1)</data_type>
  <values>A,B,C,D,F</values>
  <description>Credit rating (A=Best, F=Worst)</description>
</entity_attribute>
```

---

## Documentation Files

### Architecture Documentation

**Location:** `/docs/mdg/architecture.md`

**Contents:**
- Hub vs co-deployment decision rationale
- System landscape diagram (DEV/QA/PROD)
- Integration architecture (source systems, target systems)
- Security architecture (authentication, authorization)

---

### Domain Design Documentation

**Location:** `/docs/mdg/domain_design.md`

**Contents:**
- Domain selection (Customer, Material, Supplier)
- Data model per domain (attributes, relationships)
- Business rules per domain
- Workflow design per domain

---

### Transport Plan

**Location:** `/docs/mdg/transport_plan.md`

**Contents:**
- Transport route (DEV → QA → PROD)
- Transport schedule (when to promote)
- Transport dependencies
- Rollback procedure

---

## Script Files

### Data Migration Scripts

**Location:** `/scripts/mdg/data_migration/`

**Purpose:** Extract, transform, load legacy data into MDG

**Example:**
```python
# /scripts/mdg/data_migration/extract_legacy.py
"""
Extract customer master from SAP ECC (legacy system)
Output: customers.csv
"""
import pyrfc

conn = pyrfc.Connection(
    ashost='ecc-legacy.example.com',
    sysnr='00',
    client='100',
    user='migration_user',
    passwd='***'
)

result = conn.call('RFC_READ_TABLE', QUERY_TABLE='KNA1', ...)
```

---

### Test Data Generation

**Location:** `/scripts/mdg/test_data/`

**Purpose:** Generate realistic test data for UAT

**Example:**
```python
# /scripts/mdg/test_data/generate_customers.py
"""
Generate 1000 test customers for UAT
"""
from faker import Faker

fake = Faker()

for i in range(1000):
    customer = {
        'name': fake.company(),
        'tax_id': fake.ssn(),
        'street': fake.street_address(),
        'city': fake.city(),
        'postal_code': fake.zipcode(),
        'country': 'US'
    }
    # Insert into MDG via API or direct DB insert
```

---

## Test Files

### Integration Tests

**Location:** `/tests/mdg/integration/`

**Purpose:** Validate end-to-end workflows and integrations

**Example:**
```python
# /tests/mdg/integration/test_workflow.py
import pytest
from mdg_api import create_change_request, approve_change_request

def test_customer_change_approval_workflow():
    """Test full customer change workflow: Create → Approve → Activate"""

    # Step 1: Create change request
    cr = create_change_request(
        entity='CUSTOMER',
        entity_id='0000100001',
        changes={'CREDIT_LIMIT': 1500000}
    )
    assert cr.status == 'PENDING'

    # Step 2: Approve change request
    approve_change_request(cr.id, approver='VP_FINANCE')
    assert cr.status == 'APPROVED'

    # Step 3: Verify activation
    customer = get_customer('0000100001')
    assert customer.credit_limit == 1500000
```

---

## Version Control Strategy

### Git Repository Structure

```
mdg-project/
  .git/
  .gitignore              # Ignore sensitive files (.env, credentials)
  README.md               # Project overview
  /config/                # Git-tracked
  /docs/                  # Git-tracked
  /scripts/               # Git-tracked
  /tests/                 # Git-tracked
  /transports/            # Git-tracked (TR documentation, not actual TRs)
  .env                    # NOT git-tracked (credentials)
```

---

### .gitignore Example

```gitignore
# SAP credentials
.env
credentials.yml

# SAP transport files (managed via STMS, not git)
*.cofiles
*.data

# Python
__pycache__/
*.pyc
.venv/

# IDE
.vscode/
.idea/

# Logs
*.log
/logs/

# Sensitive data
test_data/production_copy/
```

---

## File Naming Conventions

### Transport Requests

**Format:** `DEVK<number>.md`

**Example:** `DEVK900120.md`

**Contents:**
```markdown
# Transport Request: DEVK900120

## Summary
Add custom field CREDIT_RATING to Customer entity

## Files Changed
- /config/mdg/data_model/customer/CREDIT_RATING.xml

## Testing
- Unit tests: tests/unit/test_credit_rating.py
- Integration tests: tests/integration/test_customer_workflow.py

## Dependencies
- None (can be imported independently)

## Rollback
- Revert transport via STMS
- No data migration required (new field, no existing data)
```

---

### Configuration Files

**Format:** `<entity>_<config_type>.yaml`

**Examples:**
- `customer_matching_rules.yaml`
- `material_validation_rules.yaml`
- `supplier_workflow_config.yaml`

---

### Documentation Files

**Format:** `<topic>.md`

**Examples:**
- `architecture.md`
- `domain_design.md`
- `transport_plan.md`
- `user_guide.md`

---

## Project Phases and Artifacts

### Phase 1: Foundation (Weeks 1-8)

**Artifacts:**
- `/docs/mdg/architecture.md` (hub vs co-deployment decision)
- `/docs/mdg/transport_plan.md` (DEV/QA/PROD landscape)
- `/docs/mdg/user_guide/steward_guide.md` (draft)

---

### Phase 2: Domain Implementation (Weeks 9-24)

**Artifacts per domain:**
- `/config/mdg/data_model/<domain>/` (entity extensions)
- `/config/mdg/brf_rules/<domain>_change/` (workflow rules)
- `/config/mdg/matching_rules/<domain>/` (match rules)
- `/docs/mdg/domain_design.md` (updated per domain)

---

### Phase 3: Integration (Weeks 25-32)

**Artifacts:**
- `/config/mdg/integrations/mdi/` (MDI configs)
- `/config/mdg/integrations/idoc/` (IDoc distribution models)
- `/tests/mdg/integration/test_integration.py` (integration tests)

---

### Phase 4: Testing & Go-Live (Weeks 33-40)

**Artifacts:**
- `/scripts/mdg/data_migration/` (migration scripts)
- `/tests/mdg/performance/` (performance tests)
- `/docs/mdg/cutover_plan.md` (go-live procedure)
- `/docs/mdg/rollback_plan.md` (rollback procedure)

---

## Maintenance and Updates

### Quarterly Activities

1. **Review transport dependencies** (update `transport_dependencies.md`)
2. **Archive old change requests** (reduce database size)
3. **Refresh QA client from PROD** (realistic test data)
4. **Update documentation** (reflect production changes)
5. **Review matching rules** (tune based on steward feedback)

---

### Annual Activities

1. **Architecture review** (hub sizing, performance optimization)
2. **Security audit** (access control, authorization review)
3. **Data quality review** (DQM validation accuracy)
4. **Integration review** (MDI/IDoc performance, error rates)
5. **User training refresh** (new stewards, workflow changes)
