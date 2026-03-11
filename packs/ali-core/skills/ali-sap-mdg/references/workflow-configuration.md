# ali-sap-mdg - Workflow Configuration Reference

This reference contains detailed BRFplus workflow configuration guidance, including decision table structures and code examples.

---

## Three Decision Tables Detail

Each change request type requires three decision tables configured via **USMD_SSW_RULE**:

### 1. DT_SINGLE_VAL_\<type\> (Process Flow)

**Purpose:** Define step sequence between workflow stages

**Configuration:**
```
Step 1: Create → Step 2: Validate → Step 3: Approve → Step 4: Activate
```

**Table Structure:**
- Input: Change request type, entity type
- Output: Next step in workflow sequence
- Logic: Maps current step to next step

### 2. DT_USER_AGT_GRP_\<type\> (Agent Determination)

**Purpose:** Determine who receives work items at each step

**Configuration based on:**
- Role assignments
- Organizational attributes (sales org, country, plant)
- Dynamic rules (value ranges, conditional logic)

**Table Structure:**
- Input: Change request attributes (sales org, country, amount)
- Output: Agent group ID (SAP user group)
- Logic: IF/THEN rules mapping attributes to responsible groups

### 3. DT_CONDITIONS_\<type\> (Conditional Routing)

**Purpose:** Apply IF/THEN business rules for routing decisions

**Configuration examples:**
- IF amount > $10K, require VP approval
- IF customer country = high-risk, add compliance review step
- IF material type = finished goods, add quality check

**Table Structure:**
- Input: Change request attributes
- Output: Additional approval steps, modified routing
- Logic: Complex conditional logic

---

## Change Request Types

| Type | Description | Example |
|------|-------------|---------|
| **Create** | New master record | Create new customer, new material |
| **Change** | Update existing record | Update customer address, change material price |
| **Block** | Block/unblock record | Block customer for credit issues |
| **Delete** | Soft delete (flag as deleted) | Mark supplier as inactive |
| **Mass Processing** | Bulk operations | Update 500 customers to new sales org |

---

## Example BRFplus Configurations

### Scenario 1: Credit Limit Approval

**Business Rule:** Customer change requests over $1M credit limit require VP approval

```abap
* Decision Table: DT_CONDITIONS_CUSTOMER_CHANGE
* Transaction: USMD_SSW_RULE

CONDITION_RULE: CREDIT_LIMIT_APPROVAL
  IF change_request.CREDIT_LIMIT > 1000000
  THEN
    additional_approver = 'VP_FINANCE'
    approval_level = '2'  " Two-level approval
  ELSE
    additional_approver = ''
    approval_level = '1'  " Single-level approval
```

### Scenario 2: Agent Determination by Geography

**Business Rule:** Route customer change requests to regional managers based on sales org and country

```abap
* Decision Table: DT_USER_AGT_GRP_CUSTOMER_CHANGE

AGENT_RULE: CUSTOMER_APPROVER
  IF change_request.SALES_ORG = '1000'
    AND change_request.COUNTRY = 'US'
  THEN
    agent_group = 'CUST_MGR_US_EAST'
  ELSE IF change_request.SALES_ORG = '2000'
  THEN
    agent_group = 'CUST_MGR_EUROPE'
  ELSE IF change_request.SALES_ORG = '3000'
    AND change_request.COUNTRY IN ('CN', 'JP', 'KR')
  THEN
    agent_group = 'CUST_MGR_APAC'
  ELSE
    agent_group = 'CUST_MGR_DEFAULT'  " Fallback
```

### Scenario 3: Multi-Step Approval for High-Risk Changes

**Business Rule:** Material master changes to finished goods in production plants require quality manager approval

```abap
* Decision Table: DT_CONDITIONS_MATERIAL_CHANGE

CONDITION_RULE: QUALITY_APPROVAL
  IF material.TYPE = 'FERT'  " Finished goods
    AND plant.PRODUCTION_FLAG = 'X'
    AND (change_includes('BOM') OR change_includes('ROUTING'))
  THEN
    additional_step = 'QUALITY_REVIEW'
    agent_group = 'QUALITY_MANAGERS'
    approval_required = 'X'
  ELSE
    additional_step = ''
    approval_required = ''
```

### Scenario 4: Conditional Routing Based on Value Range

**Business Rule:** Supplier changes with payment terms >90 days require CFO approval

```abap
* Decision Table: DT_CONDITIONS_SUPPLIER_CHANGE

CONDITION_RULE: PAYMENT_TERMS_APPROVAL
  IF supplier.PAYMENT_TERMS_DAYS > 90
  THEN
    additional_approver = 'CFO'
    notification = 'Extended payment terms require CFO approval'
    priority = 'HIGH'
  ELSE IF supplier.PAYMENT_TERMS_DAYS > 60
  THEN
    additional_approver = 'DIR_PROCUREMENT'
    notification = 'Payment terms >60 days require director approval'
    priority = 'MEDIUM'
  ELSE
    additional_approver = ''
    priority = 'NORMAL'
```

---

## Workflow Testing Checklist

Before promoting BRFplus configurations to production:

- [ ] All approval paths tested (approve, reject, send back)
- [ ] Agent determination tested for all combinations (org units, countries)
- [ ] Conditional routing tested for boundary values (e.g., $999,999 vs $1,000,001)
- [ ] Mass processing workflows tested with >100 records
- [ ] Error handling tested (invalid agent group, missing approver)
- [ ] Performance tested (workflow processing time <2 seconds per change request)
- [ ] Email notifications tested (correct recipients, message formatting)
- [ ] Work item display tested in NWBC/Fiori (correct information shown)

---

## Common Configuration Mistakes

| Mistake | Impact | Prevention |
|---------|--------|------------|
| **Missing fallback agent group** | Change request stuck if no match | Always define ELSE clause with default agent |
| **Circular workflow** | Step A → B → A (infinite loop) | Draw workflow diagram, validate no cycles |
| **Hardcoded user IDs** | Breaks when user leaves | Use agent groups, not individual users |
| **Incomplete conditions** | Missing edge cases (null values, zero amounts) | Test with boundary values and nulls |
| **No error handling** | Cryptic failure messages | Add explicit error conditions with clear messages |
| **Performance issues** | Complex BRFplus logic >5 seconds | Profile decision tables, simplify logic |
